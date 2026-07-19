# mooncake-p2p-store（P2P 存储）

## 1. 定位

`mooncake-p2p-store` 是 Mooncake 的**去中心化 peer-to-peer 临时对象存储**：在集群内 peer 节点之间临时共享大块内存对象（典型场景为 checkpoint / 模型权重分发），**无中心化 master 节点**，全局元数据由外部 metadata 服务（etcd / `P2PHANDSHAKE`）维护，数据搬移直接复用 [Transfer Engine](../02-transfer-engine/index.md) 的 C API。

与中心化 [`mooncake-store`](../03-store/index.md) 的关键区别：

| 维度 | mooncake-store（中心化） | mooncake-p2p-store（去中心化） |
|------|--------------------------|--------------------------------|
| 架构 | master + client 数据面，有 segment/replica/eviction/HA | client-only，无 master |
| 定位 | 长期管理的分布式 KVCache / 权重存储池 | peer 间临时对象共享（BitTorrent 式 seeding） |
| 元数据 | Master 维护（可 HA：etcd / K8s Lease） | etcd 直接 KV（`mooncake/checkpoint/` 前缀）或 P2P 握手 |
| 复制 | `replica` + `transfer_task` 主从搬移 | `Register` 做种、`GetReplica` 拉取并自动转为新数据源 |
| 语言 | C++ + Python/Go/Rust 绑定 | Go（cgo 调 TE 的 `transfer_engine_c.h`） |

**生产演进**：仓库内 `mooncake-p2p-store` 为原型实现；官方高性能生产版已独立为 [checkpoint-engine](https://github.com/MoonshotAI/checkpoint-engine/)（Sept 10, 2025 开源），已在 Kimi K1.5 / K2 生产训练中应用，跨数千 GPU 在 **~20s** 内完成 Kimi-K2（1T 参数）的权重更新。本目录代码因而定位为 P2P 原型与 TE Go 绑定的参考实现。

## 2. 目录 / 源码结构

```
mooncake-p2p-store/
├── CMakeLists.txt              仅声明 build_p2p_store 自定义目标, DEPENDS transfer_engine
├── build.sh                    实际构建入口: go build 产出 p2p-store-example
└── src/
    ├── p2pstore/               Go 包（P2P Store 核心）
    │   ├── core.go             P2PStore 结构与 Register/GetReplica/performTransfer
    │   ├── metadata.go         etcd 元数据 + Payload/Shard/Location 模型
    │   ├── catalog.go          本地已注册对象目录（内存态）
    │   ├── registered_memory.go   本地内存注册与引用计数（调 TE registerLocalMemory）
    │   ├── transfer_engine.go  cgo 封装 TE C API (transfer_engine_c.h)
    │   ├── error.go · go.mod
    └── example/
        └── p2p-store-example.go   trainer/inferencer 示例程序
```

构建开关：顶层 `CMakeLists.txt:192-195` 以 `WITH_P2P_STORE`（默认 OFF）控制 `add_subdirectory(mooncake-p2p-store)`。该子目录 `CMakeLists.txt` 本身不编译 C++，仅注册一个 `build_p2p_store` custom target，实际动作下放到 `build.sh`：以 `go build` 链接已构建的 `libtransfer_engine` / `libmooncake_common` 等，产出可执行 `p2p-store-example`（当前仅支持 RDMA 协议）。

## 3. 核心概念

### 对象模型：Payload / Shard / Location

一个共享对象（`Payload`）由若干 `Shard` 组成，每个 `Shard` 持有：

- `Gold`：原始"做种"位置（Register 节点的 `SegmentName + Offset`）。
- `ReplicaList`：经 `GetReplica` 拉取后新增的副本位置；副本节点反过来可被其它节点选作数据源。

`SegmentName` 即 TE 的 segment（peer 的 `local_server_name`，`ip:port`），`Offset` 为该 segment 内已注册内存偏移。元数据以 JSON 序列化写入 etcd，key 形如 `mooncake/checkpoint/<name>`，所有写操作用 etcd 事务 + `ModRevision` 做乐观并发控制。

### BitTorrent 式数据流

- `Register(name, addrList, sizeList, maxShardSize, ...)`：把本地内存切片为 shard，向 etcd 写入 `Gold` 位置；**不发生任何数据传输**，仅注册元数据。等价于 BitTorrent 做种。
- `GetReplica(name, addrList, sizeList)`：从 etcd 读取 payload，对每个 shard 调 `performTransfer`：随机选 `Gold` / `ReplicaList` 中的一个位置作为源，经 TE `allocateBatchID` → `submitTransfer`（`OPCODE_READ`）→ `getTransferStatus` 拉取；失败按 shard 副本数 retry。拉取完成后把本地位置写回 `ReplicaList`，**使自己成为新一轮的数据源**，从而分摊出向带宽、避免单点饱和。
- `Unregister` / `DeleteReplica`：清除 `Gold` / 本节点 `ReplicaList` 并释放本地注册内存。
- 分片粒度：`MAX_CHUNK_SIZE = 4GiB`（`core.go:30`），`maxShardSize` 推荐 64MB，需整除 `MAX_CHUNK_SIZE`（`registered_memory.go:42`）。

### P2P 握手模式

`NewTransferEngine` 的 `metadataConnString` 可传 `"P2PHANDSHAKE"`：此时 RPC 端口动态分配，peer 间通过握手发现彼此，无需 etcd。`GetLocalServerName` / `GetLocalIpAndPort`（`transfer_engine.go:210`）用于在握手后取回实际监听地址广播给 peer。

## 4. 交叉引用

- 底层传输：[`../02-transfer-engine/index.md`](../02-transfer-engine/index.md) —— `transfer_engine.go` 通过 cgo 直接调用 `transfer_engine_c.h`（`createTransferEngine` / `installTransport` / `allocateBatchID` / `submitTransfer` / `openSegment`），TE 是 P2P Store 唯一的搬移底座。
- 中心化对照：[`../03-store/index.md`](../03-store/index.md) —— 同样基于 TE，但走 master + segment + replica + task_manager 的中心化路径。
- 官方用户向文档：[`../../docs/source/design/p2p-store.md`](../../docs/source/design/p2p-store.md) —— API 签名与示例用法。
- 生产演进：[checkpoint-engine](https://github.com/MoonshotAI/checkpoint-engine/) —— 生产版权重/checkpoint 分发（README "Sept 10, 2025" 条目）。
- 顶层构建入口：`CMakeLists.txt:192`（`WITH_P2P_STORE` 开关，见 [`../00-overview/dependency-graph.md`](../00-overview/dependency-graph.md) 构建开关矩阵）。

## 5. 关键源码入口

- `mooncake-p2p-store/src/p2pstore/core.go:33` —— `P2PStore` 结构（catalog/memory/metadata/transfer 四组件）。
- `mooncake-p2p-store/src/p2pstore/core.go:57` —— `NewP2PStore`：建 metadata、建 TE、`installTransport("tcp"|"rdma")`、装 RegisteredMemory。
- `mooncake-p2p-store/src/p2pstore/core.go:123` —— `Register`：本地内存切片为 shard，写 etcd `Gold`。
- `mooncake-p2p-store/src/p2pstore/core.go:329` —— `GetReplica`：读 payload → `doGetReplica` 并发拉取 → 写回 `ReplicaList`。
- `mooncake-p2p-store/src/p2pstore/core.go:364` —— `performTransfer`：单 shard 的 TE 批传输 + retry（`allocateBatchID` / `submitTransfer` / `getTransferStatus`）。
- `mooncake-p2p-store/src/p2pstore/metadata.go:51` —— `Shard` / `Payload` / `Location` JSON 模型与 etcd 事务。
- `mooncake-p2p-store/src/p2pstore/metadata.go:114` —— `NewMetadata`（etcd 客户端，`METADATA_KEY_PREFIX = "mooncake/checkpoint/"`，`core.go:31`）。
- `mooncake-p2p-store/src/p2pstore/transfer_engine.go:36` —— `NewTransferEngine`（cgo `createTransferEngine`）。
- `mooncake-p2p-store/src/p2pstore/registered_memory.go:28` —— `RegisteredMemory` 本地注册与引用计数。
- `mooncake-p2p-store/src/example/p2p-store-example.go:41` —— `p2p-store-example` 入口（`--cmd=trainer|inferencer`）。
- `mooncake-p2p-store/build.sh:77` —— `go build -o p2p-store-example` 实际构建命令。
- `mooncake-p2p-store/CMakeLists.txt:2` —— `build_p2p_store` 目标 `DEPENDS transfer_engine`。
- `CMakeLists.txt:192` —— 顶层 `WITH_P2P_STORE` gate（默认 OFF）。

## 参考文档

- 官方设计：[`../../docs/source/design/p2p-store.md`](../../docs/source/design/p2p-store.md)
- 主仓库 README P2P Store 条目：[`../../README.md`](../../README.md)（Sept 10, 2025）
- Transfer Engine Go 绑定说明：`../../docs/source/design/transfer-engine/index.md`（P2P Store 运行需求）
