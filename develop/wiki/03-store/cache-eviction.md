# 缓存与驱逐

## 一、定位

本页覆盖 `mooncake-store` 的**热度缓存**、**内存/SSD 驱逐机制**与**对象保护**三层。Store 既在 master 侧维护全局元数据驱动的批量驱逐（控制内存/NoF 池水位），又在 client 侧提供本地热缓存（`LocalHotCache`）以减少跨节点 Get。驱逐需绕开 pin 保护与引用计数，并按租户配额公平回收。本页与 [replication-scheduling.md](replication-scheduling.md) 的 pin/refcnt 字段紧密相关。

## 二、目录 / 源码结构

```
mooncake-store/
├── include/
│   ├── local_hot_cache.h        LocalHotCache / HotMemBlock / HotCachePutToken / Handler
│   ├── eviction_strategy.h      EvictionStrategy 抽象 + LRU/FIFO 实现
│   ├── count_min_sketch.h       CountMinSketch 频率估计 (admission gating)
│   ├── hybrid_metric.h          hybrid_counter / hybrid_histogram (静态+动态标签)
│   ├── tenant_quota.h           TenantQuotaTable / Reserve/Commit/Release
│   ├── tenant_quota_policy_store.h  YAML/etcd 策略存储
│   ├── pinned_buffer_pool.h     PinnedBufferPool 复用 pinned host 内存
│   └── pinned_host_buffer.h     PinnedHostBuffer RAII 句柄
├── src/
│   ├── local_hot_cache.cpp      热缓存实现 (LRU/SHM/token)
│   ├── master_service.cpp       EvictionThreadFunc / BatchEvict / NoFBatchEvict / 租户驱逐
│   ├── tenant_quota.cpp         配额分配与 effective quota 计算
│   └── tenant_quota_policy_store.cpp  YAML/etcd 策略加载
```

## 三、核心概念

### 1. 全局驱逐线程 (`EvictionThreadFunc`)

`master_service.cpp:5648` 的后台线程周期性（`kEvictionThreadSleepMs`）检查全局内存使用率 `get_global_mem_used_ratio()`。触发条件：使用率超过 `eviction_high_watermark_ratio_`，或 `need_mem_eviction_` 被置位（分配失败时置位于 `master_service.cpp:2923`）。触发后计算目标回收比例 `evict_ratio_target = max(eviction_ratio_, used - high_watermark + eviction_ratio_)` 与下界 `evict_ratio_lowerbound`，交给 `BatchEvict` 执行。NoF 池走对称的 `NoFBatchEvict` 配 `nof_eviction_*` 水位。低活跃期则改做 `DiscardExpiredProcessingReplicas`（清理过期 PROCESSING 副本与残留复制任务）。

### 2. BatchEvict 两阶段并行驱逐

`BatchEvict` 采用两阶段并行（`master_service.cpp:6565`）：

- **Phase 1 候选采集**：N 个线程各扫一批 shard，随机起始 shard 避免倾斜，收集 lease 过期且 `can_evict_replicas`（无 refcnt 占用）的候选 key。`IsHardPinned()` 直接跳过；soft-pinned 在 `allow_evict_soft_pinned_objects_` 开启时另行收集。
- **Phase 2 驱逐**：按候选的 `lease_timeout` 排序，依次 `try_evict_or_offload`。驱逐分两趟：第一趟 `allow_soft_pinned=false`，若未达目标且策略允许，再跑 `allow_soft_pinned=true` 兜底（`master_service.cpp:6335`）。

分组对象（`group_id`）按组驱逐：仅当组内全部成员 lease 过期才整组回收（`try_evict_group_or_object`，`master_service.cpp:6491`），保证组内元数据/驱逐语义一致。

### 3. pin 保护与引用计数

对象级保护由 [replication-scheduling.md](replication-scheduling.md) 的 `ReplicateConfig` 字段落地到 `ObjectMetadata`：

- `hard_pinned`（`master_service.h:929`）：创建时 immutable，永不驱逐，upsert 自动继承（`master_service.cpp:3726`）。
- `soft_pin`（`soft_pin_timeout`，`master_service.h:927`）：带 TTL 的软保护，仅在过期或 `allow_evict_soft_pinned_objects_` 开启时可驱逐；TTL 可经 lease 续期（`GrantLease`）。
- `refcnt_`（`Replica::refcnt_`，`replica.h:586`）：副本级原子引用计数，复制/迁移/读路径 `inc_refcnt()` 保护源副本不被驱逐，完成后 `dec_refcnt()`（如 `CopyStart`，`master_service.cpp:4036/4106`）。`can_evict_replicas` 检查 refcnt 为零才纳入候选。

### 4. offload-on-evict

当 `enable_disk_eviction_` 与 offload-on-evict 模式开启时，驱逐内存副本时不是直接丢弃，而是将其降级 (`offload`) 到 LOCAL_DISK，由 `LocalDiskSegment::offloading_objects` 承接（`segment.h:95`）。这使驱逐转化为“内存→本地盘”的层级下沉，而非数据消失，后续命中可经 promotion-on-hit 提升回内存。

### 5. promotion-on-hit 与 Count-Min Sketch

读路径命中仅 LOCAL_DISK 的 key 时触发提升（`PushPromotionQueue`，`master_service.h:1520`），但受频率准入控制：`promotion_sketch_`（`CountMinSketch`，`master_service.h:1946`）对每次命中 `increment(admission_key)`（`master_service.cpp:5249`），并叠加 watermark / dedup / cap 门限。`CountMinSketch`（`count_min_sketch.h:14`）以 `depth × width` 的 `uint8` 表存多哈希最小估计，累计达 `width*depth` 自动 `decay`（整体右移 1）防饱和。该机制以 O(1) 空间近似过滤冷 key，避免冷对象反复提升抖动。

### 6. LocalHotCache —— 客户端本地热缓存

`LocalHotCache`（`local_hot_cache.h:47`）是 client 侧的块式 LRU 热缓存，按 `block_size_`（默认 16MB）从 bulk 内存切块，可选 `use_shm` 经 `memfd` 创建跨进程共享段供 dummy client mmap 复用。特性：

- **deferred LRU touch**：`HotMemBlock::accessed` 标志延迟 touch，批量 `drainDeferredTouches` 批量搬到队首，减少锁竞争。
- **async fill + token**：`LocalHotCacheHandler` 以 worker 线程异步填充。`HotCachePutToken{cache_epoch, key_generation}` 在提交时校验，若期间 key 被 `RemoveHotKey` / `BumpKeyGeneration` / `Clear` 失效则取消发布、归还 block，避免悬空填充。
- **in-use 保护**：`GetHotKey` 标记 block 为 in_use，`ReleaseHotKey` 释放，防止驱逐正在被读的块。
- **批量与正则淘汰**：`RemoveHotKeys` / `RemoveHotKeysByRegex` 支持批量与模式匹配清理。

### 7. EvictionStrategy 基础策略

`EvictionStrategy`（`eviction_strategy.h:16`）抽象 key 序的 `AddKey/UpdateKey/EvictKey`，提供 `LRUEvictionStrategy`（带头插、尾驱逐）与 `FIFOEvictionStrategy` 两个基础实现，供需要 key 级 LRU/FIFO 的局部场景使用（master 全局驱逐走 BatchEvict 分片扫描而非此抽象）。

### 8. 租户配额 (`TenantQuotaTable`)

`TenantQuotaTable`（`tenant_quota.h:53`）维护每租户 `TenantQuotaState`：`requested_quota_bytes` / `effective_quota_bytes` / `used_bytes` / `reserved_bytes`。`BuildEffectiveQuotaAssignments` 将各租户请求按 `allocatable_capacity_bytes` 归一化为生效配额（`RecomputeEffectiveQuotas`）。写入流程 `Reserve → Commit/Abort`，删除走 `Release/ReleasePartial`，超额置 `over_quota`。`TenantQuotaPolicyStore`（`tenant_quota_policy_store.h:26`）支持 YAML 文件与 etcd（`STORE_USE_ETCD`）两种后端持久化策略。租户配额也参与驱逐：`TenantQuotaEvictionResult`（`master_service.cpp:6238`）在配额回收时按租户维度选择可驱逐对象。

### 9. PinnedBufferPool —— D2H 加速缓冲池

`PinnedBufferPool`（`pinned_buffer_pool.h:25`）是线程安全的 pinned host 内存复用池：`Acquire` 按容量从池中取，无则经 `AcceleratorDevice::AllocatePinnedHost` 新分配（pinned 内存 D2H 带宽较 pageable 高 10x~100x），分配失败回退 `new char[]`。`Release` 在池满（`kDefaultMaxPoolSize=32`）时立即释放而非缓存，防止 pinned 内存无限增长。`PinnedHostBuffer`（`pinned_host_buffer.h:10`）为 RAII 句柄，持有 `addr/size/deleter`。

### 10. hybrid_metric

`hybrid_metric.h` 定义 `basic_hybrid_counter` / `basic_hybrid_histogram`（基于 ylt::metric），支持**静态标签 + 动态标签**混合：构造时给定一组静态 label（预格式化），运行时按动态 label 数组 inc/observe。驱逐与配额相关的运行指标（如按租户/介质的回收字节数、命中率）由此类组件暴露给 Prometheus/Grafana。

## 四、交叉引用

- pin / refcnt 字段定义与副本生命周期见 [replication-scheduling.md](replication-scheduling.md)。
- 驱逐产生的对象变更经 KV 事件发布，见 [kv-event-serialize.md](kv-event-serialize.md)；驱逐线程的启停与 master HA 主备切换相关，见 [metadata-ha.md](metadata-ha.md)。
- 官方 SSD 卸载设计：`../../docs/source/design/ssd-offload.md`；HiCache 分层缓存（Store 作为外部存储后端）：`../../docs/source/design/hicache-design.md`；Engram 索引与提升路由相关：`../../docs/source/design/engram.md`。

## 五、关键源码入口

| 入口 | 定位 |
|------|------|
| `EvictionThreadFunc` 驱逐主循环 | `mooncake-store/src/master_service.cpp:5648` |
| `BatchEvict` 两阶段并行驱逐 | `mooncake-store/src/master_service.cpp:6490` |
| soft/hard pin 在 upsert 中的继承 | `mooncake-store/src/master_service.cpp:3722` |
| 分配失败触发 `need_mem_eviction_` | `mooncake-store/src/master_service.cpp:2923` |
| `Replica::refcnt_` 保护与 inc/dec | `mooncake-store/include/replica.h:586` |
| CopyStart refcnt 保护源副本 | `mooncake-store/src/master_service.cpp:4036` |
| `CountMinSketch` 频率估计 | `mooncake-store/include/count_min_sketch.h:14` |
| promotion-on-hit 频率准入 | `mooncake-store/src/master_service.cpp:5249` |
| `LocalHotCache` 客户端热缓存 | `mooncake-store/include/local_hot_cache.h:47` |
| `HotCachePutToken` 异步填充校验 | `mooncake-store/include/local_hot_cache.h:27` |
| `EvictionStrategy` LRU/FIFO | `mooncake-store/include/eviction_strategy.h:16` |
| `TenantQuotaTable` 配额管理 | `mooncake-store/include/tenant_quota.h:53` |
| 租户维度的驱逐选择 | `mooncake-store/src/master_service.cpp:6238` |
| `PinnedBufferPool` pinned 内存池 | `mooncake-store/include/pinned_buffer_pool.h:25` |
| `hybrid_metric` 混合标签指标 | `mooncake-store/include/hybrid_metric.h:11` |
