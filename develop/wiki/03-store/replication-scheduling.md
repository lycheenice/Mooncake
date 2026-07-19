# 复制与调度

## 一、定位

本页覆盖 `mooncake-store` 中**对象写入时的副本布局**与**数据搬移调度**两条主线：前者决定一个对象的多个副本落在哪些 Segment 上（含大对象 striping、副本数与 pin 策略），后者决定实际字节流如何从 Client 搬到这些 Segment（含策略选择、并行 I/O、deadline 定时与任务队列）。底层字节搬移统一走 TransferEngine API，故本页是 Store 与 TE 的主要交叉点之一。

## 二、目录 / 源码结构

```
mooncake-store/
├── include/
│   ├── replica.h                Replica/ReplicateConfig/ReplicaWriteMode/ReplicaStatus
│   ├── replica_selection.h      读路径副本选择 + 远程打分钩子 (issue #2516)
│   ├── allocation_strategy.h    AllocationStrategy 抽象与多种实现 + AllocatorManager
│   ├── segment.h                SegmentManager / 挂载状态 / RAII Scoped*Access
│   ├── transfer_task.h          TransferStrategy / TransferFuture / TransferSubmitter / WorkerPool
│   ├── task_manager.h           ClientTaskManager REPLICA_COPY/MOVE 任务与 RAII 访问器
│   ├── thread_pool.h            通用 ThreadPool
│   ├── deadline_scheduler.h     模板化 DeadlineScheduler (min-heap 定时器)
│   ├── offset_allocator/
│   │   └── offset_allocator.h   OffsetAllocator (bin-based, RAII Handle, 可序列化)
│   └── types.h                  kMaxSliceSize / AllocationStrategyType / ReplicaType
├── src/
│   ├── transfer_task.cpp        TransferSubmitter 实现 + 各 WorkerPool
│   ├── task_manager.cpp         任务生命周期 + msgpack/zstd HA 序列化
│   ├── thread_pool.cpp          ThreadPool 实现
│   ├── offset_allocator.cpp     OffsetAllocator 包装实现
│   ├── segment.cpp              SegmentManager 挂载/卸载/分配门面
│   ├── master_service.cpp       副本分配主流程 (PutStart/UpsertStart)
│   └── client_buffer.cpp        split_into_slices 大对象切片
└── tests/                       allocation_strategy_test · replica_selection_test ·
                                 deadline_scheduler_test · offset_allocator_test
```

## 三、核心概念

### 1. ReplicateConfig —— 对象级复制策略

`ReplicateConfig`（`replica.h:81`）是每次写入携带的策略包，字段含义：

| 字段 | 作用 |
|------|------|
| `replica_num` | MEMORY 副本数（默认 1） |
| `nof_replica_num` | NVMe-oF SSD 副本数（默认 0） |
| `with_soft_pin` / `with_hard_pin` | 软/硬 pin，阻止驱逐；hard pin 使对象完全不可驱逐 |
| `preferred_segments` | 偏好段名列表，分配优先尝试 |
| `preferred_nof_segments` | NoF 段的偏好列表 |
| `prefer_alloc_in_same_node` | 倾向同节点分配 |
| `host_id` / `group_ids` | 写者主机（用于 local-first 排序）与按 key 分组的元数据/驱逐分组 |

据此 `DetermineReplicaWriteMode`（`replica.h:156`）将写入归为三档：`SINGLE_REPLICA`、`FLEXIBLE_DUAL_REPLICA`（1 内存 + 1 NoF，可降级）、`RELIABLE_MULTI_REPLICA`（多副本强一致）。`FLEXIBLE_DUAL_REPLICA` 模式下若内存分配失败可继续而非整体失败（`master_service.cpp:2921`）。

### 2. Replica 生命周期与多介质

`Replica`（`replica.h:209`）以 `std::variant` 承载四种介质数据：`MemoryReplicaData` / `NoFReplicaData` / `DiskReplicaData` / `LocalDiskReplicaData`，对应 `ReplicaType` 枚举（`MEMORY` / `NOF_SSD` / `DISK` / `LOCAL_DISK`）。状态机 `ReplicaStatus`（`replica.h:51`）为 `UNDEFINED → INITIALIZED → PROCESSING → COMPLETE`，另有 `REMOVED` / `FAILED`。`refcnt_` 原子引用计数用于驱逐保护（见 [cache-eviction.md](cache-eviction.md)）。每个 Replica 全局唯一 `ReplicaID` 由原子自增分配。

### 3. 大对象 striping

单个对象被切成多个 Slice，每片上限 `kMaxSliceSize`（`types.h:479`，取自 CacheLib `Slab::kSize - 16`）。`split_into_slices`（`client_buffer.cpp:132`）按该上限循环切片。每个 Slice 在 master 侧独立走分配流程，从而一个大对象的多片可落到不同 Segment 实现条带化（striping），且每片各自持有 `replica_num` 个副本——这是 Store 的并行 I/O 基础：多 Slice × 多副本构成可并发提交的传输请求集合。

### 4. AllocationStrategy —— 副本布局策略

`AllocationStrategy`（`allocation_strategy.h:140`）是抽象接口，`Allocate()` 返回 `std::vector<Replica>`，采用 **best-effort 语义**：若无法满足全部请求副本数，则尽量多分配，仅当一片都分不到时才返回 `NO_AVAILABLE_HANDLE`。关键保证是**同一 Slice 的多个副本必落不同 Segment**。实现族由 `CreateAllocationStrategy`（`allocation_strategy.h:774`）按 `AllocationStrategyType`（`types.h:501`）产出：

| 实现 | 策略 |
|------|------|
| `RandomAllocationStrategy` | 随机选段，偏好段优先，同 Slice 副本分簇 |
| `FreeRatioFirstAllocationStrategy` | 采样候选段按空闲比降序取 top-N，近最优负载均衡，回退随机 |
| `SsdFreeRatioFirstAllocationStrategy` | SSD 介质的空闲比优先，依赖 `SsdMetricsProvider` |
| `CxlAllocationStrategy` | 强制走指定 CXL 段 |

`AllocatorManager`（`allocation_strategy.h:26`）按段名管理多个 `BufferAllocatorBase`，提供随机挑选能力；其线程安全由 `SegmentManager::segment_mutex_` 经 `ScopedAllocatorAccess`（`segment.h:322`）保证。master 在 `PutStart` 流程（`master_service.cpp:2909`）中先注入 `preferred_segments` 与 `host_id` 的 local-first 段序，再调用 `Allocate()`。

### 5. TransferSubmitter —— 搬移策略选择与并行 I/O

`TransferSubmitter`（`transfer_task.h:529`）是写入/读取的提交器。`submit()` 分析 `Replica::Descriptor` 与 Slices，经 `selectStrategy()`（`transfer_task.cpp:1348`）选四种 `TransferStrategy`（`transfer_task.h:32`）之一：

- `LOCAL_MEMCPY`：同进程端点（`isSameProcessEndpoint`，要求 ip:port 完全一致而非仅同主机，`transfer_task.cpp:1395`），走 `MemcpyWorkerPool` 异步 memcpy；
- `TRANSFER_ENGINE`：跨节点走 `TransferEngine` 的 batch API（`submitTransferEngineOperation`），这是 Store 调用 TE 的核心入口；
- `SPDK_NVMF`：`USE_NOF` 下走 `SpdkNofWorkerPool`，带 inflight 限流（`SpdkNofQos`）；
- `FILE_READ`：磁盘副本走 `FilereadWorkerPool`。

`submit_batch()`（`transfer_task.cpp:1038`）将多个 Replica 的多 Slice 聚合提交，形成并行 I/O。所有操作返回 `TransferFuture`（`transfer_task.h:254`），底层 `OperationState` 子类封装各策略的等待逻辑（condition_variable / TE 轮询），提供 `isReady()` / `wait()` / `get()` future 语义。

### 6. 读路径副本选择

读路径由 `SelectBestReplica`（`replica_selection.h:122`）决定优先级：local MEMORY > local NOF_SSD > remote MEMORY > LOCAL_DISK > DISK。远程 MEMORY 多副本时，可经 `MC_STORE_REPLICA_SCORING=1` 或注入 `ReplicaScorer`（`SetRemoteReplicaScorer`，`replica_selection.h:65`）启用打分，内置打分偏好 RDMA 优于 TCP。该设计使 Store 可接收来自 TE 的拓扑/负载信号而不直接依赖 TE 层。

### 7. TaskManager —— 副本迁移任务队列

`ClientTaskManager`（`task_manager.h:196`）管理 `REPLICA_COPY` / `REPLICA_MOVE` 两类异步任务（`TaskType`，`task_manager.h:16`），payload 为 `ReplicaCopyPayload` / `ReplicaMovePayload`（JSON 序列化）。任务经 `PENDING → PROCESSING → SUCCESS/FAILED` 流转，由 `ScopedTaskReadAccess` / `ScopedTaskWriteAccess`（RAII 持锁）操作，含 pending/processing/finished 三级队列、超时清理（`prune_expired_tasks`）、重试上限与 LRU 完成任务回收。`TaskManagerSerializer`（`task_manager.h:254`）以 msgpack + zstd 压缩做 HA 快照序列化。

### 8. DeadlineScheduler 与 ThreadPool

`DeadlineScheduler<Id>`（`deadline_scheduler.h:16`）是模板化 min-heap 定时器：单 timer 线程以 `condition_variable::wait_until` 等待最近 deadline，到期回调在外释放锁以防锁序反转，支持 `RemoveIf` 批量撤销。Store 中实例化为 `DeadlineScheduler<GracefulUnmountDeadlineRecord>`（`master_service.h:1511`）驱动 Segment 优雅卸载倒计时。通用 `ThreadPool`（`thread_pool.h:19`）为固定 worker 数的任务队列，供需要并发执行的非传输类工作复用。

### 9. offset 分配

`OffsetAllocator`（`offset_allocator/offset_allocator.h:137`）是对 Sebastian Aaltonen bin-based 分配器（`__Allocator`）的线程安全包装：按 size class 分桶、容量的动态伸缩、支持超出最大桶（3.75GB）的大块，并暴露碎片化指标（`OffsetAllocatorMetrics`）。`OffsetAllocationHandle` 为 RAII 句柄，析构自动归还。分配器状态可经 `serialize_to` / `deserialize_from` 模板做二进制快照，用于 master HA fork/snapshot 恢复段内分配状态。

## 四、交叉引用

- 底层搬移依赖 TransferEngine：副本跨节点传输走 TE batch API，详见 [`../02-transfer-engine/index.md`](../02-transfer-engine/index.md)。
- 副本驱逐与 pin 保护（`with_soft_pin` / `with_hard_pin` / `refcnt_`）由缓存层处理，见 [`cache-eviction.md`](cache-eviction.md)。
- 任务状态与分配器状态的 HA 序列化复用 [`kv-event-serialize.md`](kv-event-serialize.md) 的序列化框架，并参与 [metadata-ha.md](metadata-ha.md) 的快照/oplog。
- 官方架构概览：`../../docs/source/design/architecture.md`；Store 设计：`../../docs/source/design/mooncake-store.md`；SSD 空闲比策略：`../../docs/source/design/ssd-free-ratio-first-allocation.md`；SSD 卸载：`../../docs/source/design/ssd-offload.md`。

## 五、关键源码入口

| 入口 | 定位 |
|------|------|
| `ReplicateConfig` 定义 | `mooncake-store/include/replica.h:81` |
| `DetermineReplicaWriteMode` | `mooncake-store/include/replica.h:156` |
| `Replica` 状态机与多介质 variant | `mooncake-store/include/replica.h:209` |
| `AllocationStrategy` 抽象与实现族 | `mooncake-store/include/allocation_strategy.h:140` |
| `CreateAllocationStrategy` 工厂 | `mooncake-store/include/allocation_strategy.h:774` |
| PutStart 副本分配主流程 | `mooncake-store/src/master_service.cpp:2909` |
| 大对象切片 `split_into_slices` | `mooncake-store/src/client_buffer.cpp:132` |
| `kMaxSliceSize` 常量 | `mooncake-store/include/types.h:479` |
| `TransferSubmitter::submit` 策略分发 | `mooncake-store/src/transfer_task.cpp:981` |
| `selectStrategy` 本地/TE 判定 | `mooncake-store/src/transfer_task.cpp:1348` |
| `isSameProcessEndpoint` 同进程判定 | `mooncake-store/src/transfer_task.cpp:1395` |
| `SelectBestReplica` 读路径选择 | `mooncake-store/include/replica_selection.h:122` |
| `ClientTaskManager` 任务管理 | `mooncake-store/include/task_manager.h:196` |
| 任务超时清理 `prune_expired_tasks` | `mooncake-store/src/task_manager.cpp:163` |
| `DeadlineScheduler` 模板 | `mooncake-store/include/deadline_scheduler.h:16` |
| 优雅卸载定时器实例 | `mooncake-store/include/master_service.h:1511` |
| `ThreadPool` 通用线程池 | `mooncake-store/include/thread_pool.h:19` |
| `OffsetAllocator` 偏移分配器 | `mooncake-store/include/offset_allocator/offset_allocator.h:137` |
