# KV 事件与序列化

## 一、定位

本页覆盖 `mooncake-store` 的两条数据导出通路：**(1) KV 事件发布**——master 将对象写入/删除/驱逐变更以 RFC #1527 标准化事件经 ZMQ PUB 推送给外部索引器（如 Conductor）；**(2) 内部序列化框架**——master 状态（Segment/Replica/Allocator/TaskManager）的二进制与 msgpack 表示，用于 HA 快照、oplog 与 fork。两者共同构成 Store 与外部消费者及备节点之间的数据契约层。

## 二、目录 / 源码结构

```
mooncake-store/
├── include/
│   ├── kv_event/
│   │   ├── kv_event_publisher.h   KvEventPublisher (ZMQ PUB, 异步批处理)
│   │   ├── kv_event_config.h      KvEventConfig (传输/身份/兼容开关)
│   │   └── key_util.h             ParseSeqHashFromObjectKey (decimal/0x hex u64)
│   ├── serializer.h               两趟序列化框架 (SizeCounter/Writer/Reader)
│   └── serialize/
│       └── serializer.h           Serializer<T> msgpack 特化 (Allocator/Buffer/Replica/Segment)
├── src/
│   ├── kv_event/
│   │   └── kv_event_publisher.cpp ZMQ bind + worker 线程 + msgpack 打包
│   ├── serialize/
│   │   └── serializer.cpp         Serializer<T> 各特化实现
│   ├── master_service.cpp         集成：PublishKvStored/Removed + BuildKvEventConfig
│   ├── task_manager.cpp           TaskManagerSerializer (msgpack + zstd)
│   └── ha/oplog/
│       ├── oplog_serializer.cpp   OpLogEntry JSON 序列化
│       └── localfs_oplog_store.cpp 本地 fs oplog 持久化
└── CMakeLists.txt                 option(ENABLE_KV_EVENTS) (默认 OFF, 需 libzmq)
```

## 三、核心概念

### 1. KvEventPublisher —— 对象变更事件发布

`KvEventPublisher`（`kv_event_publisher.h:22`）在 `ENABLE_KV_EVENTS=ON`（需 libzmq，`src/CMakeLists.txt:121`）时编译为真实实现，否则编译为 no-op stub（`kv_event_publisher.h:93`），使 master 在无 ZMQ 环境下零开销跳过。构造时（`kv_event_publisher.cpp:70`）创建 ZMQ context、PUB socket（设 `ZMQ_SNDHWM=10000`、`ZMQ_LINGER=0`）、`zmq_bind` 到 `bind_endpoint`，并起 worker 线程。

对外仅两个发布接口：

- `PublishStored(object_key, medium, tenant_id, group_id)`（`kv_event_publisher.cpp:141`）
- `PublishRemoved(...)`（`kv_event_publisher.cpp:152`）

二者均**非阻塞**：将 `PendingEvent` 压入有界 `deque`（`queue_capacity` 默认 65536），满时丢最旧并预留一个 ZMQ sequence gap（`Enqueue`，`kv_event_publisher.cpp:172`），使消费者可检测丢事件。worker 线程（`WorkerLoop`，`kv_event_publisher.cpp:204`）批取出至多 `kMaxBatchSize=64` 条，经 `PublishBatch` 打包发送；析构时 `DrainRemainingQueue` 排空剩余。

### 2. Wire 格式与 RFC #1527 契约

`PublishBatch`（`kv_event_publisher.cpp:225`）发送三帧 ZMQ 消息，与 vLLM/SGLang 一致：

1. 空 topic 帧；
2. big-endian `uint64` 序列号（`next_zmq_sequence_` 自增，丢事件时也自增以保 gap）；
3. msgpack payload = `[timestamp_ms, [events...], dp_rank]`（`kv_event_publisher.cpp:255`）。

每个 event 是 msgpack map，字段遵循 RFC #1527（见 `../../docs/source/design/conductor/indexer-api-design.md`）：

- 信封：`event_id`、`timestamp`、`event_type`("stored"/"removed")、`model_name`/`block_size`/`additional_salt`/`lora_name`/`dp_rank`（master 不掌握，置 nil，由索引器 `POST /register` 补齐）、`tenant_id`、`backend_id`、`medium`、`group_id`；
- `seq_hashes`：由 `ParseSeqHashFromObjectKey`（`key_util.h:10`）尝试把 object key 解析为 decimal 或 `0x` hex `u64`；解析成功则发单元素数组，否则空数组并（在 `emit_object_key` 开启时）附 `object_key` 供 sha256+key 匹配；
- stored 事件额外 `base_block_idx=0`、`parent_hash=nil`、`token_ids=nil`（master 把每个 key 当独立 pool block，深度 0 满足 RFC 的 base_block_idx/parent_hash 存在性要求）；
- 兼容开关：`emit_legacy_compat_fields`（默认 on）追加 `type`("BlockStored"/"BlockRemoved")、`block_hashes`、`parent_block_hash` 供 Dynamo relay 直转；`emit_object_key`（默认 on）追加 `object_key`。

统计 `Stats`（published/dropped/skipped）经 `GetStats` 暴露，master 经 `GetKvEventStats`（`master_service.cpp:8801`）上报指标。

### 3. master 集成与触发点

master 在构造时 `BuildKvEventConfig`（`master_service.cpp:8706`）把 `MasterServiceConfig` 的 `enable_kv_events` / `kv_events_bind_endpoint` / `kv_events_backend_id` 等映射为 `KvEventConfig`，并 `make_unique<KvEventPublisher>`（`master_service.cpp:402`）。事件触发点：

- 写入完成 `PutEnd` → `PublishKvStored(key, replica_type, metadata, tenant_id)`（`master_service.cpp:3309`）；
- 显式删除 / 对象失效 → `PublishKvRemoved`（`master_service.cpp:3899`）；
- 驱逐 → `PublishKvRemovedAfterEvict`（`master_service.cpp:6540` 等），介质为 `"cpu"` / `"disk"`。

`MediumForReplicaType`（`master_service.cpp:8724`）将 `MEMORY→"cpu"`、`DISK/LOCAL_DISK/NOF_SSD→"disk"`；`MediumForMetadata` 按是否含内存副本决定。master 仅发 `cpu`/`disk` 两介质，永不发 `gpu`（GPU 副本由推理引擎自发布）。

### 4. 两趟序列化框架 (`serializer.h`)

`serializer.h` 提供通用二进制序列化抽象，要求被序列化类实现 `serialize_to(T&)` 模板与 `static deserialize_from(T&)`：

- `SerializeSizeCounter`（`serializer.h:74`）：第一趟只累加大小不写内存；
- `SerializeWriter`（`serializer.h:152`）：第二趟写入预分配 buffer，带越界/空指针检查；
- `SerializerReader`（`serializer.h:246`）：反序列化读，越界抛 `runtime_error`；
- 顶层 `serialize_to<T>` / `deserialize_from<T>` 模板（`serializer.h:301/353`）串起两趟并校验 `finish_write`/`finish_read`。

该框架用于 master HA 的二进制快照（如 `OffsetAllocator` 自带 `serialize_to` 模板，见 [replication-scheduling.md](replication-scheduling.md)）。

### 5. msgpack `Serializer<T>` 特化 (`serialize/serializer.h`)

`serialize/serializer.h` 声明一组 `Serializer<T>` 模板特化，以 msgpack 为编码，覆盖 Store 核心可快照对象：

| 特化 | 用途 |
|------|------|
| `Serializer<__Allocator>` / `Serializer<OffsetAllocator>` / `Serializer<OffsetAllocationHandle>` | 段内偏移分配器状态（见 [replication-scheduling.md](replication-scheduling.md)） |
| `Serializer<AllocatedBuffer>` | 副本底层缓冲描述符 |
| `Serializer<Replica>` | 单副本（含其 buffer） |
| `Serializer<MountedSegment>` | 挂载段（含其 `OffsetBufferAllocator`） |
| `Serializer<OffsetBufferAllocator>` | 段级分配器 |

实现见 `serialize/serializer.cpp`。这些特化供 `SegmentSerializer` / `MasterSnapshotManager` 组合调用，把 master 全部 Segment+Replica+Allocator 状态序列化为一个快照，用于 HA fork、snapshot 传输与 standby 恢复。

### 6. TaskManagerSerializer 与 oplog

- `TaskManagerSerializer`（`task_manager.h:254`，实现 `task_manager.cpp:291`）把 `ClientTaskManager` 的全部 `Task`（按 task_id 排序保证确定性）以 msgpack 打包后用 **zstd** 压缩（级别 3），反序列化时 zstd 解压 + msgpack unpack + `restore_task`。它参与 HA 快照，使复制/迁移任务队列在主备切换后可恢复。
- HA oplog 走另一套：`ha/oplog/oplog_serializer.cpp` 把 `OpLogEntry` 序列化为 JSON 字符串，`localfs_oplog_store.cpp` 以 `[uint32 length][serialized_value]` 定长前缀格式追加写入本地 fs，用于 master 操作日志的持久化与重放。

两类序列化各司其职：msgpack/zstd 用于全量快照（紧凑、批量），JSON 用于 oplog（可读、易调试）。

### 7. 序列化用途汇总

| 用途 | 格式 | 涉及代码 |
|------|------|----------|
| HA 全量快照（Segment/Replica/Allocator/Task） | msgpack（+ zstd for Task） | `serialize/serializer.cpp` · `task_manager.cpp` · `master_snapshot*` |
| HA oplog 重放 | JSON | `ha/oplog/oplog_serializer.cpp` · `localfs_oplog_store.cpp` |
| RPC 元数据传输 | `YLT_REFL` 反射（struct_json） | `rpc_types.h` 各 struct 的 `YLT_REFL` 宏 |
| 外部 KV 事件发布 | msgpack over ZMQ | `kv_event/kv_event_publisher.cpp` |

## 四、交叉引用

- KV 事件由 Conductor 等外部索引器消费，发布者/消费者关系见 [engram-conductor.md](engram-conductor.md)；wire 协议规范见 `../../docs/source/design/conductor/indexer-api-design.md`。
- 快照 / oplog / standby 的 HA 流程见 [metadata-ha.md](metadata-ha.md)；序列化对象（Replica/Segment）的定义见 [replication-scheduling.md](replication-scheduling.md)。
- 驱逐触发 `PublishKvRemovedAfterEvict`，见 [cache-eviction.md](cache-eviction.md)。
- 官方 KV 事件设计：`../../docs/source/design/conductor/indexer-api-design.md`；Store master 发布者章节见同文 “Mooncake Store master publisher”。

## 五、关键源码入口

| 入口 | 定位 |
|------|------|
| `KvEventPublisher` 类（真实实现） | `mooncake-store/include/kv_event/kv_event_publisher.h:22` |
| no-op stub（无 ZMQ 时） | `mooncake-store/include/kv_event/kv_event_publisher.h:93` |
| `KvEventConfig` 字段 | `mooncake-store/include/kv_event/kv_event_config.h:12` |
| `ParseSeqHashFromObjectKey` | `mooncake-store/include/kv_event/key_util.h:10` |
| ZMQ bind + worker 启动 | `mooncake-store/src/kv_event/kv_event_publisher.cpp:70` |
| 有界队列 + 丢旧 + sequence gap | `mooncake-store/src/kv_event/kv_event_publisher.cpp:172` |
| 三帧 wire 格式打包 | `mooncake-store/src/kv_event/kv_event_publisher.cpp:225` |
| master 构造 publisher | `mooncake-store/src/master_service.cpp:402` |
| `BuildKvEventConfig` | `mooncake-store/src/master_service.cpp:8706` |
| `PublishKvStored`（PutEnd 触发） | `mooncake-store/src/master_service.cpp:8750` |
| `MediumForReplicaType` 介质映射 | `mooncake-store/src/master_service.cpp:8724` |
| `ENABLE_KV_EVENTS` 构建开关 | `mooncake-store/CMakeLists.txt:3` · `mooncake-store/src/CMakeLists.txt:121` |
| 两趟序列化框架 | `mooncake-store/include/serializer.h:74` |
| `Serializer<T>` msgpack 特化 | `mooncake-store/include/serialize/serializer.h:27` |
| `Serializer<T>` 实现 | `mooncake-store/src/serialize/serializer.cpp:21` |
| `TaskManagerSerializer` (msgpack+zstd) | `mooncake-store/src/task_manager.cpp:291` |
| oplog JSON 序列化 | `mooncake-store/src/ha/oplog/oplog_serializer.cpp:36` |
| 本地 fs oplog 存储格式 | `mooncake-store/src/ha/oplog/localfs_oplog_store.cpp:395` |
