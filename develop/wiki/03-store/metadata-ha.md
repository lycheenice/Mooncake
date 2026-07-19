# Store 元数据后端与高可用（HA）

## 定位

本页覆盖 `mooncake-store` 的两件事：

1. **元数据后端选择**：Master 的元数据（`mooncake/ram/*`、`mooncake/rpc_meta/*`、OpLog）落在哪个外部存储上。可选 `etcd` / `Redis` / 自带 `HTTP metadata server` / K8s Lease。
2. **HA 高可用**：Master 主备切换、OpLog 复制、Snapshot 恢复、Standby 状态机。让单点 Master 变成可故障转移的 leader/follower 拓扑。

二者紧密耦合：`HA` 的 leader election 与 OpLog 持久化都构建在上述元数据后端之上，且**构建期互斥约束**决定了同一进程只能链接一种 Go c-shared 后端。

## 目录 / 源码结构

```
mooncake-store/
├── 元数据层
│   ├── etcd_helper.cpp/h              etcd c-shared 客户端封装（CreateWithLease/Watch ...）
│   ├── k8s_lease_helper.cpp/h         K8s Lease leader election 封装（RunElection/WatchHolder/SetPodLabel）
│   ├── http_metadata_server.cpp/h     内嵌 coro_http 元数据 KV 服务（可独立部署）
│   ├── metadata_store.h               Standby 侧 metadata 抽象接口 + StandbyObjectMetadata
│   └── serialize/                     元数据序列化器（snapshot 编解码）
├── HA（src/ha/）
│   ├── common/redis/                  Redis 后端公共件
│   ├── leadership/
│   │   ├── master_service_supervisor.cpp   进程级监督循环（candidate → leader/standby）
│   │   ├── leader_coordinator_factory.cpp  按 backend 造 LeaderCoordinator
│   │   └── backends/{etcd,k8s,redis}/       三种 leader election 后端实现
│   ├── oplog/
│   │   ├── etcd_oplog_store.cpp           EtcdOpLogStore（OpLog 写入 etcd）
│   │   ├── localfs_oplog_store.cpp        LocalFsOpLogStore（本地 FS，测试/单机）
│   │   ├── oplog_manager.cpp              OpLogManager（序列号分配 / 缓冲）
│   │   ├── oplog_replicator.cpp           Standby 侧拉取 + 应用循环
│   │   ├── oplog_applier.cpp              把 OpLogEntry 翻译成 master 元数据变更
│   │   ├── oplog_serializer.cpp           struct_pack 二进制编解码
│   │   ├── oplog_store_factory.cpp        按 config 选 OpLogStore 实现
│   │   ├── etcd_oplog_change_notifier.cpp etcd watch 推送
│   │   └── polling_oplog_change_notifier.cpp 轮询式推送（兼容后端）
│   ├── snapshot/
│   │   ├── catalog_backed_snapshot_provider.cpp  Standby 基线加载实现
│   │   ├── master_snapshot_codec.cpp            MasterService 状态 ↔ snapshot payload 编解码
│   │   ├── catalog/backends/                    snapshot 目录存储（embedded / redis）
│   │   └── object/backends/                     snapshot payload 存储（local / s3）
│   └── standby_controller.cpp             StandbyController：装配 HotStandbyService + 状态映射
├── hot_standby_service.cpp/h             HotStandbyService：Standby 主体（replication loop / promote）
├── standby_state_machine.cpp/h           StandbyStateMachine：9 态状态机
├── ha_metric_manager.cpp/h               HA 专用 metric
├── master_snapshot_manager.cpp/h         Primary 侧周期 snapshot 编排（fork child）
└── master_snapshot_repository.cpp/h      snapshot 产物仓库
```

公共的 `etcd` / `k8s-lease` Go c-shared 库源自 `mooncake-common/`，Store 只做薄封装。

## 核心概念

### 元数据后端四选

| 后端 | 开关 | 用途 | 备注 |
|------|------|------|------|
| etcd | `STORE_USE_ETCD` | 元数据 KV + OpLog 存储 + leader election | 默认 HA 后端（`FLAGS_ha_backend_type="etcd"`） |
| Redis | `STORE_USE_REDIS` | leader election / snapshot catalog | 需 `hiredis` |
| HTTP | `FLAGS_enable_http_metadata_server` | 内嵌 coro_http KV 元数据服务 | 可与 master 同进程或独立部署 |
| K8s Lease | `STORE_USE_K8S_LEASE` | K8s 原生 leader election + Pod label route | 与 etcd 互斥 |

`metadata_server` 在 `RealClient`/`MasterClient` 中通过 `MOONCAKE_TE_META_DATA_SERVER` 或 `MOONCAKE_CONFIG_PATH` 的 `metadata_server` 字段解析；`P2PHANDSHAKE` 特殊值表示无中心元数据。当 client TTL 超时，master 可经 `setHttpMetadataServer` / `setHttpMetadataRemoteUrl` 对 co-located 或独立部署的 HTTP server 做 `mooncake/ram/*` 清理。

### 构建互斥约束

`STORE_USE_K8S_LEASE` 与 `STORE_USE_ETCD` **不能同时开启**，因为两者都构建 Go c-shared HA 后端库，同进程双链会冲突：

```cmake
# CMakeLists.txt:49-58
option(STORE_USE_K8S_LEASE "..." OFF)
if (STORE_USE_K8S_LEASE)
  if (STORE_USE_ETCD)
    message(FATAL_ERROR "STORE_USE_K8S_LEASE and STORE_USE_ETCD cannot be enabled together ...")
  endif()
  ...
endif()
```

运行期由 `ValidateHABackendAvailability`（`ha_types.h:55-80`）按 `#ifdef STORE_USE_*` 把未编译的后端返回 `UNAVAILABLE_IN_CURRENT_MODE`。

### MasterServiceSupervisor 监督循环

`MasterServiceSupervisor::Start`（`master_service_supervisor.cpp`）跑一个常驻循环：

1. `EnterStandbyMode`：先以 Standby 启动 `StandbyController`，开始追随当前 leader 的 OpLog。
2. `ReadCurrentView`：从 `LeaderCoordinator` 读当前 leader，无 leader 则 `TryAcquireLeadership` 竞选。
3. 竞选成功 → `WarmupLeadership`（在 lease TTL 内续约）→ `ActivateServingState`（`SetServiceAvailable(true)` + 设 route label）。
4. 失去 leader → `DeactivateServingState` → 回到 Standby。

K8s 后端额外由 `LeaderLabelReconciler` 调 `K8sLeaseHelper::SetPodLabel` 给 leader Pod 打 `mooncake.io/store-role=leader`，供 service route 用。

### StandbyStateMachine 状态机

9 态（`standby_state_machine.h:49-76`）：

```
STOPPED → CONNECTING → SYNCING → WATCHING ⇄ RECOVERING/RECONNECTING
                                            ↓ Promote()
                                          PROMOTING → PROMOTED
                                            ↓ fatal
                                          FAILED
```

仅 `WATCHING` 态可 `Promote()`。`ProcessEvent` 校验迁移合法性、记录 `TransitionRecord` 历史、回调 `StateChangeCallback`。`kMaxConsecutiveErrors=10`、`kMaxReconnectAttempts=100` 为容限常量。

### OpLog 复制

`OpLogStore`（`oplog_store.h`）抽象 OpLog 持久层，实现：`EtcdOpLogStore`、`LocalFsOpLogStore`。流程：

1. **Primary** 侧 `OpLogManager` 分配递增 `sequence_id`，序列化后 `WriteOpLog` 落 backend。
2. **Standby** 侧 `OpLogChangeNotifier`（etcd watch 或轮询）感知新条目，`OpLogReplicator` 拉取 → `OpLogApplier` 把 `OpLogEntry` 翻译成 `MetadataStore`（`StandbyMetadataStore`）上的 Put/Remove。
3. `RecordSnapshotSequenceId` / `GetSnapshotSequenceId` 记录 snapshot 截止 sequence，供清理与对齐。

OpLog following 仅 etcd 后端开启（`spec.type == ETCD`，见 `standby_controller.cpp:41`）；K8s/Redis 后端走 snapshot-only 模式。

### Snapshot bootstrap 与恢复

Standby 启动时优先 `LoadSnapshotBaselineLocked` 取最新 snapshot（`SnapshotProvider` / `LoadedSnapshot`），把 `applied_seq_id` 推到 `snapshot_sequence_id`，再 `StartOplogFollowingLocked` 重放之后的 OpLog。Snapshot 由 Primary 侧 `MasterSnapshotManager` 周期性 fork child 生成，经 `MasterSnapshotCodec` 编码，payload 与 catalog 分别落 `object/backends/`（local / s3）和 `catalog/backends/`（embedded / redis）。恢复路径：`RestoreState` → `ApplySnapshotState`。

### Promotion（接任 Primary）

`HotStandbyService::Promote`：先 `FinalCatchUpForPromotionLocked` 把 lag 追平，`ResolvePromotionGapsLocked` 补缺失 sequence，状态翻 `PROMOTING → PROMOTED`，`ExportMetadataSnapshot` 把 `StandbyMetadataStore` 全量导给新 Primary 的 `MasterService` 做 fast recovery。`MasterRuntimeState`（`kStandby` / `kRecovering` / `kCatchingUp` / `kLeaderWarmup` / `kServing`）由 `MapStandbyRuntimeState` 映射。

## 交叉引用

- 公共层 etcd / k8s-lease 来源：[`../01-common/index.md`](../01-common/index.md)
- 官方架构概览：[`../../docs/source/design/architecture.md`](../../docs/source/design/architecture.md)
- Store 总览：[`index.md`](index.md)
- 依赖图中 HA 横向：[`../00-overview/dependency-graph.md`](../00-overview/dependency-graph.md)（"元数据 / HA 横向"段）

## 关键源码入口

- `CMakeLists.txt:49` — `STORE_USE_K8S_LEASE` 选项与互斥 FATAL_ERROR（root `CMakeLists.txt:49-58`）
- `mooncake-store/include/ha/ha_types.h:21` — `enum HABackendType` 四后端枚举
- `mooncake-store/include/ha/ha_types.h:55` — `ValidateHABackendAvailability` 运行期可用性闸
- `mooncake-store/src/ha/leadership/master_service_supervisor.cpp:218` — `RunSupervisorLoop` 监督主循环
- `mooncake-store/src/ha/leadership/master_service_supervisor.cpp:200` — `EnterStandbyMode` Standby 入口
- `mooncake-store/include/standby_state_machine.h:49` — `enum StandbyState` 9 态状态机
- `mooncake-store/include/standby_state_machine.h:109` — `enum StandbyEvent` 触发事件集
- `mooncake-store/include/hot_standby_service.h:82` — `class HotStandbyService` Standby 主体
- `mooncake-store/include/hot_standby_service.h:98` — `HotStandbyService::Start` 启动入口
- `mooncake-store/include/hot_standby_service.h:128` — `HotStandbyService::Promote` 接任入口
- `mooncake-store/include/ha/standby_controller.h:14` — `class StandbyController` 抽象
- `mooncake-store/src/ha/standby_controller.cpp:41` — `BuildStandbyRuntimeCapabilities` OpLog following 开关
- `mooncake-store/include/ha/oplog/oplog_store.h:27` — `class OpLogStore` OpLog 持久层抽象
- `mooncake-store/include/ha/snapshot/snapshot_provider.h:34` — `class SnapshotProvider` baseline 加载
- `mooncake-store/include/metadata_store.h:68` — `class MetadataStore` Standby 侧元数据接口
- `mooncake-store/include/etcd_helper.h:16` — `class EtcdHelper` etcd 封装
- `mooncake-store/include/k8s_lease_helper.h:11` — `class K8sLeaseHelper` K8s Lease 封装
- `mooncake-store/include/http_metadata_server.h:22` — `class HttpMetadataServer` 内嵌 HTTP KV
