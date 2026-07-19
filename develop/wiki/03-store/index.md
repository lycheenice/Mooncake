# mooncake-store 子系统总览

## 定位

`mooncake-store` 是 Mooncake 中**最大的子系统**，定位为**分布式 KVCache / 模型权重存储引擎**。它把散落在各 worker 节点的异构内存（DRAM/VRAM/NVMe）抽象为统一的 Tensor 对象池，对外提供 `Get` / `Put` / `List` / `Del` / `Replicate` 语义，对内通过 `mooncake-transfer-engine` 完成跨节点、跨硬件的数据搬移。

Store 采用**控制面 / 数据面分离**架构：

- **控制面 Master**：全局元数据权威（metadata authority），维护 `key → replica 列表 → segment` 的映射，负责对象 lease、quota、驱逐决策、复制/迁移任务调度、HA 主备切换。无大数据路径，只持有元数据 shard。
- **数据面 Client**：每个 worker 节点一个 `RealClient`，挂载本地 Segment（一段已注册给 TE 的内存/VRAM/SSD 区间），执行真正的 `Put` 写入与 `Get` 读取，通过 TE 的 `transfer_task` 在 Segment 之间搬移字节。

本页是 `03-store/` 目录的入口，先俯瞰两面结构与对象生命周期数据流；元数据/HA 细节见 [`metadata-ha.md`](metadata-ha.md)，多级存储后端见 [`storage-backends.md`](storage-backends.md)。

## 目录 / 源码结构

```
mooncake-store/
├── 控制面 Master
│   ├── master.cpp                        进程入口 + gflags 定义 + InitMasterConf
│   ├── master_service.cpp                MasterService 主体（元数据 shard / lease / 驱逐 / 复制）
│   ├── master_admin_service.cpp          MasterAdminServer HTTP 端点 + 运行态/route label
│   ├── master_client.cpp                 客户端桩：向 master 发 RPC（client 侧使用）
│   ├── master_metric_manager.cpp         master 侧 metric（key_count / eviction / quota）
│   ├── master_snapshot_manager.cpp       周期性 snapshot 编排（child process fork）
│   ├── master_snapshot_repository.cpp    snapshot 产物存储仓库抽象
│   ├── tenant_quota.cpp                  多租户配额账本
│   └── tenant_quota_policy_store.cpp     租户配额策略持久化
├── 数据面 Client
│   ├── real_client.cpp                   RealClient 主体（setup / mount / put / get / IPC）
│   ├── real_client_main.cpp              独立 mooncake-store-cli 进程入口
│   ├── client_service.cpp                client 间 RPC 服务（offload batch_get 等）
│   ├── client_buffer.cpp                 ClientBufferAllocator（本地 buffer 池）
│   ├── aligned_client_buffer.cpp         对齐包装的 client buffer
│   ├── dummy_client.cpp                  DummyClient：同机 IPC 复用 RealClient 的 shm
│   ├── client_metric.cpp                 client 侧 metric
│   ├── pyclient.h                        面向 Python 绑定的 PyClient 接口
│   └── store_c.cpp                       C API 导出（store_c.h）
├── Segment / 对象层
│   ├── segment.cpp                       Segment / MountedSegment 状态机
│   ├── storage_backend.cpp               本地 SSD bucket 存储（offload 落盘）
│   ├── offset_allocator/                 段内偏移分配器
│   ├── mmap_arena.cpp                    mmap 大页 arena（SGLang HiCache 复用）
│   ├── memory_alloc.cpp                  hugepage_memory_alloc/free
│   ├── allocator.cpp                     CachelibBufferAllocator / OffsetBufferAllocator
│   └── allocation_strategy.h             random / free_ratio_first / cxl / local_first 策略
├── 多级存储后端          详见 storage-backends.md
├── 复制与任务            transfer_task / task_manager / thread_pool / deadline_scheduler
├── 缓存与驱逐            local_hot_cache / eviction_strategy / count_min_sketch / hybrid_metric
├── HA                    ha/(common/leadership/oplog/snapshot/standby_controller)  详见 metadata-ha.md
├── Engram 索引           engram/engram_store（对应 docs Conductor 设计）
├── KV 事件               kv_event/kv_event_publisher（RFC #1527, ZMQ PUB）
├── 序列化 / 元数据       serialize/ · etcd_helper · k8s_lease_helper · http_metadata_server · metadata_store
├── RPC / 工具            rpc_service(+helper+types) · uds_transport · config_helper · master_config · utils · version
└── go/ · rust/ · conf/ · benchmarks · tests
```

控制面与数据面的对应头文件分别置于 `include/master_*.h` 与 `include/real_client.h` / `client_*.h` / `pyclient.h` / `store_c.h`。

## 核心概念

### 对象与 Replica

一个对象（key）由若干 `Replica` 组成，每个 `Replica` 落在某个 Segment 上，并带 `ReplicaType`（`MEMORY` / `DISK` / `LOCAL_DISK` / `NOF_SSD`）与 `ReplicaStatus`（`PROCESSING` / `COMPLETE` / `INVALID`）。元数据在 `MasterService` 中按 1024 个 shard 分片（`kNumShards`），每 shard 独立 `SharedMutex`，锁序约定见 `master_service.h:80-91`。

### Put / Get 数据流

```
Put(key, value):
  Client ── PutStart ──► Master：按 allocation_strategy 选 Segment，分配 buffer handle
  Client ◄── replica descriptors ── Master
  Client → 通过 TE 把 value 写入选中 Segment（本地或远端 worker）
  Client ── PutEnd ──► Master：将 replica 状态翻转 COMPLETE、记 lease、commit quota

Get(key):
  Client ── GetReplicaList ──► Master：返回可用 replica + lease 续期
  Client → 通过 TE 从最优 replica 拷贝到本地 buffer（replica_selection.h 评分）
  Client ── (可选) PutRevoke / Evict ──► Master
```

`Replicate` / `Copy` / `Move` 在 master 侧由 `task_manager` + `transfer_task` 编排，下发到持有源 Segment 的 client 执行实际 TE 搬移；`Offload`（DRAM→SSD）与 `Promotion`（SSD→DRAM）走心跳驱动的任务队列。

### 控制面 Master 进程

`master.cpp` 是进程入口：定义 gflags（`FLAGS_enable_ha` / `FLAGS_allocation_strategy` / `FLAGS_default_kv_lease_ttl` 等），`InitMasterConf` 把默认值/配置文件/命令行归并到 `MasterConfig`。`MasterService` 是元数据权威，`MasterAdminServer` 暴露 HTTP `/metrics`、`/health`、运行态查询；HA 模式下由 `MasterServiceSupervisor` 监督 leader election 与主备切换（见 [`metadata-ha.md`](metadata-ha.md)）。

### 数据面 Client 进程

`RealClient::setup_internal` 完成：(1) 通过 `Client::Create` 向 TE 注册本机；(2) 挂载 `global_segment_size` 的 Segment（按 `max_mr_size` 切片，支持 NUMA 分布、hugepage）；(3) 注册 `local_buffer_size` 给 TE；(4) 可选启动 IPC server（与 `DummyClient` 共享 shm）、offload RPC server、HTTP server。`ResourceTracker` 用 `atexit` + 信号线程保证资源回收，对 Python 嵌入环境特殊处理。

### Segment 生命周期

`SegmentStatus` 在 `segment.h:26-33` 定义：`OK` → `DRAINING` → `DRAINED` → `GRACEFULLY_UNMOUNTING` → `UNMOUNTING`，支撑 drain job（`CreateDrainJob`）与优雅下线。

## 交叉引用

- 官方 Store 设计：[`../../docs/source/design/mooncake-store.md`](../../docs/source/design/mooncake-store.md)
- 官方架构概览：[`../../docs/source/design/architecture.md`](../../docs/source/design/architecture.md)
- 传输引擎底座：[`../02-transfer-engine/`](../02-transfer-engine/)（`transfer_task` / `task_manager` 走 TE API）
- 公共层 etcd / k8s-lease：[`../01-common/`](../01-common/)
- 总览架构拓扑：[`../00-overview/architecture.md`](../00-overview/architecture.md)
- 依赖关系：[`../00-overview/dependency-graph.md`](../00-overview/dependency-graph.md)

本目录共 8 个模块页，其余 7 个为：

- [`metadata-ha.md`](metadata-ha.md) — 元数据后端（etcd/Redis/HTTP/K8s Lease）与 HA（主备切换 / OpLog / Snapshot）
- [`storage-backends.md`](storage-backends.md) — 多级存储后端（VRAM/DRAM/SSD/远端）
- `segment-allocator.md` — Segment 对象层与 `offset_allocator` / `allocation_strategy` 分配策略
- `replica-task-manager.md` — Replica / `transfer_task` / `task_manager` / `deadline_scheduler` 复制与任务调度
- `cache-eviction.md` — `local_hot_cache` / `eviction_strategy` / `count_min_sketch` / `hybrid_metric` 缓存与驱逐
- `engram-conductor.md` — `engr_store` 智能索引（对应 docs Conductor 设计）
- `kv-events-client-bindings.md` — `kv_event_publisher`（RFC #1527）+ `PyClient` / `store_c` 绑定层

## 关键源码入口

- `mooncake-store/src/master.cpp:113` — gflags 定义起点（`FLAGS_config_path` 等）
- `mooncake-store/src/master.cpp:173` — `DEFINE_bool(enable_ha, ...)` HA 总开关
- `mooncake-store/src/master.cpp:417` — `InitMasterConf`：默认值/配置/命令行归并入口
- `mooncake-store/include/master_service.h:92` — `class MasterService` 元数据权威主体
- `mooncake-store/include/master_service.h:139` — `MountSegment`：Segment 注册 API
- `mooncake-store/include/master_service.h:379` — `PutStart`：对象写入起点
- `mooncake-store/include/master_service.h:343` — `GetReplicaList`：对象读取起点
- `mooncake-store/src/real_client.cpp:561` — `RealClient` 构造与 hugepage 探测
- `mooncake-store/src/real_client.cpp:598` — `RealClient::setup_internal`：数据面初始化主路径
- `mooncake-store/src/real_client.cpp:953` — `RealClient::setup_real`：Python 侧入口封装
- `mooncake-store/include/segment.h:26` — `enum SegmentStatus`：Segment 生命周期状态
- `mooncake-store/include/allocation_strategy.h:26` — `AllocatorManager`：Segment 内分配器容器
- `mooncake-store/CMakeLists.txt:1` — Store 子系统构建入口（版本 / HA 后端开关）
