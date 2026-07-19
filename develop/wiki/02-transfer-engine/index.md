# 传输引擎核心（Transfer Engine Core）

## 定位

`mooncake-transfer-engine`（简称 TE）是 Mooncake 全栈的**通用数据搬移底座**。它围绕两个核心抽象——Segment（远端可寻址的连续地址空间）与 BatchTransfer（一批异步读/写请求）——在异构互连（RDMA / TCP / NVLink / EFA / NVMe-oF / 各厂商加速器直连）之上提供统一的零拷贝传输 API。Store 的 `transfer_task`、P2P / EP / PG 的底层搬移、RL 示例与 Python 绑定全部走 TE API。

TE 既是协议/硬件无关的抽象层，也是多 NIC 带宽聚合、NUMA 拓扑感知路径选择、容错重试与元数据交换的发生地。其下一代子树 `tent/`（见 [`tent.md`](tent.md)）对原 TE 做能力增强而非替换，二者在同一 `TransferEngine` 门面后并存。

## 目录 / 源码结构

```
mooncake-transfer-engine/
├── include/                       公共头（被 Store/EP/PG/integration 引用）
│   ├── transfer_engine.h          门面类 TransferEngine（聚合 impl_ / impl_tent_）
│   ├── transfer_engine_impl.h     经典实现 TransferEngineImpl
│   ├── multi_transport.h          多传输选择器 MultiTransport
│   ├── multi_transport_locality.h 段名→主机的局部性判定（hip/TCP fast-path）
│   ├── topology.h                 NUMA/设备拓扑矩阵 Topology
│   ├── transfer_metadata.h        传输元数据 TransferMetadata（SegmentDesc/BufferDesc/HandShake）
│   ├── transfer_metadata_plugin.h 元数据存储 & 握手插件接口
│   ├── config.h                   GlobalConfig（RDMA/QP/slice/重试等全局参数）
│   ├── memory_location.h          NUMA 物理位置探测 + segments 编码
│   ├── graceful_shutdown.h        优雅退出 ShutdownToken
│   ├── show_links.h               NIC 诊断信息导出
│   ├── transport/transport.h      Transport 基类（TransferRequest/Slice/Task/BatchDesc）
│   └── (分配器 / gpu_vendor 见 allocators.md)
├── src/
│   ├── transfer_engine.cpp        TransferEngine 门面，按 USE_TENT 分两条实现路径
│   ├── transfer_engine_impl.cpp   经典引擎初始化/内存注册/notify/metrics
│   ├── transfer_engine_c.cpp      C ABI（供 Rust/Go/Python FFI）
│   ├── multi_transport.cpp        submitTransfer→selectTransport 调度核心
│   ├── topology.cpp               拓扑发现/解析/设备选择
│   ├── transfer_metadata.cpp      元数据编解码/缓存/refresh poller
│   ├── transfer_metadata_plugin.cpp  etcd/redis/http 插件 + TCP 握手守护
│   ├── transfer_metadata_dump.cpp 元数据调试导出
│   ├── config.cpp                 环境变量→GlobalConfig 加载
│   ├── graceful_shutdown.cpp      信号处理 + ShutdownToken 注册
│   ├── memory_location.cpp        /proc/numa_maps 扫描
│   └── show_links.cpp             NicDiagInfo 聚合
├── transport/                     传输后端插件族（见 transports.md）
├── tent/                          下一代 TENT（见 tent.md）
└── rust/ · benchmark/ · example/ · scripts/ · tests/
```

## 核心概念

### 1. 门面与双后端

`TransferEngine`（`include/transfer_engine.h:52`）是唯一对外类，内含两个实现指针：

```cpp
std::shared_ptr<TransferEngineImpl>      impl_;       // 经典 TE
std::shared_ptr<mooncake::tent::TransferEngine> impl_tent_; // TENT
bool use_tent_{false};
```

`transfer_engine.cpp` 用 `#ifndef USE_TENT ... #else ... #endif` 在编译期决定门面方法委托给 `impl_` 还是 `impl_tent_`（经典路径见 `transfer_engine.cpp:102`，TENT 路径见 `transfer_engine.cpp:435`）。上层调用者无需感知后端切换。

### 2. Segment / Buffer / Batch

- **Segment**：以 `local_server_name` 命名的远端可寻址地址空间，逻辑覆盖整个进程地址空间。`openSegment(name)` 返回 `SegmentHandle`（即 `SegmentID`）。元数据由 `TransferMetadata::SegmentDesc` 描述（`transfer_metadata.h:106`），含 devices/topology/buffers/nvmeof_buffers/cxl/rank_info 等多协议字段。
- **Buffer**：Segment 内一段连续、同设备的注册区，携带 RDMA `lkey/rkey`、shm 名、cxl 偏移等。`BufferDesc` 见 `transfer_metadata.h:56`。
- **Batch**：`allocateBatchID(size)` 返回不透明 `BatchID`（实为 `BatchDesc*` 重解释，见 `transport.h:102`）。`submitTransfer(batch_id, entries)` 把多个 `TransferRequest`（READ/WRITE + source + target_id + target_offset + length）入队，异步轮询 `getTransferStatus`。

### 3. MultiTransport —— 传输选择

`MultiTransport`（`multi_transport.h:25`）持 `transport_map_<proto, Transport>`。`submitTransfer`（`multi_transport.cpp:115`）对每个 request 调 `selectTransport`（`multi_transport.cpp:452`）：

1. 取目标 `SegmentDesc`，读其 `protocol` 字段。
2. 单协议段直接按 proto 取对应 Transport。
3. 多协议段（`protocol` 含逗号，如 `"rdma,hip"`）按目标地址落在哪个 buffer、再按固定优先级选最高性能 transport（`mp_selectTransport`，`multi_transport.cpp:544`）。
4. 同主机且仅 TCP 时回退 local memcpy（`isTcpOnly()`，`multi_transport.cpp:611`）。

`installTransport(proto, topo)`（`multi_transport.cpp:314`）是后端工厂，按 `#ifdef USE_*` 实例化 `RdmaTransport / TcpTransport / EfaTransport / NvlinkTransport / HipTransport / AscendDirectTransport ...` 等。

### 4. 拓扑感知路径选择

`Topology`（`topology.h:64`）维护一张 `TopologyMatrix`：对每种存储类型（`cpu:0`/`gpu:0`/...）列出 preferred 与 avail HCA 列表。`discover()`（`topology.h:74`）扫描本机 RDMA 设备与 NUMA 亲和，`resolve()` 构建按本地 HCA 索引的 peer 亲和表。`selectDevice(storage_type, hint)（`topology.h:92`）在 preferred 列表中轮询选 NIC，失败时回退 avail 列表。

请求 >64KB 被 Transport 内部切成 slice，每个 slice 可走不同 NIC 路径，从而**多 NIC 带宽聚合**。`enable_hca_peer_affinity` / `nic_peer_affinity`（`config.h:87-88`）允许按 peer 固定 NIC 映射，避免跨 UPI/PCIe 开销。

### 5. 元数据与插件机制

`TransferMetadata`（`transfer_metadata.h:45`）管理 segment/buffer/rpc/notify 的本地缓存与远端同步。它持两个插件：

- **MetadataStoragePlugin**（`transfer_metadata_plugin.h:21`）：KV 存储，`Create(conn_string)` 按 scheme 分派到 etcd/redis/http 实现。`get/set/remove` 三个原语。
- **HandShakePlugin**（`transfer_metadata_plugin.h:33`）：TCP socket 守护，负责 QP 握手、metadata exchange、notify、probe 四类 RPC。`startDaemon` / `send` / `exchangeMetadata` / `registerOn*CallBack`。

引擎 init 时根据 `metadata_conn_string`（如 `etcd://host:2379`）自动构造插件。`startHandshakeDaemon`（`transfer_metadata.h:237`）在后台监听握手端口（默认 12001，`config.h:50`）。`te_metadata_refresh_interval_seconds`（`config.h:70`）开启后台 refresh poller 周期性同步远端 segment 缓存。

### 6. 容错与重试

- **slice 级重试**：`retry_cnt`（`config.h:53`，默认 9）控制 RDMA slice 失败重试次数。
- **rail 冷却**：`rdma_rail_pause_seconds`（`config.h:64`）在 rail 失败后暂停该 rail 一段时间。
- **连接熔断**：`conn_pause_ttl_ms`（`config.h:81`）在 endpoint 拆除后暂停向该 peer 地址主动重连，避免 k8s rolling restart 场景下阻塞 posting worker。
- **slice 超时**：`slice_timeout`（`config.h:73`）在 `getTransferStatus` 中检查（`multi_transport.cpp:211`），超时转 `TIMEOUT`。
- **peer 存活探针**：`probePeerAliveByID`（`transfer_engine.h:160`）经 `sendProbe` 探测远端。
- **优雅退出**：`enableGracefulShutdown()`（`transfer_engine.h:204`）注册 `ShutdownToken`，在信号到达时 `freeEngine()` 释放资源（`graceful_shutdown.h`）。

### 7. 多协议内存注册

`ENABLE_MULTI_PROTOCOL` 下，`mp_registerLocalMemory`（`transfer_engine.h:126`）允许同一批 buffer 按 protocol 分类注册，`mp_submitTransfer`（`transfer_engine.h:134`）返回实际使用的 proto。典型场景：device KV pool 同时注册到 `rdma` 和 `hip`，由 selector 按可达性选最优（节点内 hip / 跨节点 rdma，判定见 `multi_transport_locality.h:74`）。

### 8. 与 Store/EP 的 device transport 钩子

CUDA/MUSA/MACA 构建下，`TransferEngine` 额外暴露 `getOrCreateP2pTransport` / `getOrCreateRdmaTransport`（`transfer_engine.h:175`），供 EP 取节点内 P2P 与 IBGDA（GPUDirect Async）device transport，懒创建并由引擎持有。

## 交叉引用

- 官方设计文档：[`../../docs/source/design/transfer-engine/index.md`](../../docs/source/design/transfer-engine/index.md)（Segment / BatchTransfer / Topology Aware / Endpoint Management 全貌）
- 官方 C++ API：[`../../docs/source/design/transfer-engine/cpp-api.md`](../../docs/source/design/transfer-engine/cpp-api.md)
- 传输后端族：[`transports.md`](transports.md)
- 内存分配器：[`allocators.md`](allocators.md)
- 下一代 TENT：[`tent.md`](tent.md)
- 硬件协议矩阵：[`../00-overview/hardware-protocol-matrix.md`](../00-overview/hardware-protocol-matrix.md)
- 架构拓扑与 TE 定位：[`../00-overview/architecture.md`](../00-overview/architecture.md)
- 依赖关系（TE ← Store/EP/PG/integration）：[`../00-overview/dependency-graph.md`](../00-overview/dependency-graph.md)

## 关键源码入口

| 关注点 | 入口 |
|--------|------|
| 引擎初始化 | `mooncake-transfer-engine/src/transfer_engine_impl.cpp:77`（`TransferEngineImpl::init`） |
| 门面 init（经典/TENT 分支） | `mooncake-transfer-engine/src/transfer_engine.cpp:102` 与 `:435` |
| 内存注册 | `mooncake-transfer-engine/src/transfer_engine_impl.cpp:586`（`registerLocalMemory`） |
| 多协议内存注册 | `mooncake-transfer-engine/src/transfer_engine_impl.cpp:648`（`mp_registerLocalMemory`） |
| 提交传输 | `mooncake-transfer-engine/src/multi_transport.cpp:115`（`MultiTransport::submitTransfer`） |
| 传输选择 | `mooncake-transfer-engine/src/multi_transport.cpp:452`（`selectTransport`） / `:544`（`mp_selectTransport`） |
| 后端工厂 | `mooncake-transfer-engine/src/multi_transport.cpp:314`（`installTransport`） |
| 拓扑发现/选设备 | `mooncake-transfer-engine/src/topology.cpp`（`Topology::discover` / `selectDevice`，头在 `include/topology.h:74,92`） |
| 元数据握手守护 | `mooncake-transfer-engine/src/transfer_metadata_plugin.cpp`（`HandShakePlugin::Create` / `startDaemon`） |
| 元数据缓存 refresh | `mooncake-transfer-engine/src/transfer_metadata.cpp`（`startMetadataRefreshPolling`，头 `include/transfer_metadata.h:263`） |
| 全局配置加载 | `mooncake-transfer-engine/src/config.cpp`（`loadGlobalConfig`，头 `include/config.h:137`） |
| 优雅退出注册 | `mooncake-transfer-engine/src/graceful_shutdown.cpp`（`installGracefulShutdownHandlers`，头 `include/graceful_shutdown.h:37`） |
| NUMA 位置探测 | `mooncake-transfer-engine/src/memory_location.cpp`（`getMemoryLocation`，头 `include/memory_location.h:37`） |
| NIC 诊断 | `mooncake-transfer-engine/src/show_links.cpp`（`buildShowLinksJson`，头 `include/show_links.h:36`） |
| 状态轮询 + 超时 | `mooncake-transfer-engine/src/multi_transport.cpp:200`（`getTransferStatus`） |
| peer 存活探针 | `mooncake-transfer-engine/src/transfer_engine_impl.cpp:520`（`probePeerAliveByID`） |
| C ABI | `mooncake-transfer-engine/src/transfer_engine_c.cpp` |
| Transport 基类 | `mooncake-transfer-engine/include/transport/transport.h:44`（`TransferRequest`/`Slice`/`BatchDesc`） |
