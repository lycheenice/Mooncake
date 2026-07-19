# Mooncake 架构拓扑

## 一句话定位

Mooncake 是一个 **KVCache 为中心的 disaggregated LLM 推理 / 训练基础设施**，以 **Tensor 作为全栈基础数据载体**：从异构存储（DRAM/VRAM/NVMe）间的高效数据搬移，到分布式 Tensor 对象（KVCache、模型权重）的管理，再到基于 Tensor 的弹性分布式计算，贯穿推理与训练全链路。

## 顶层子系统树

```
Mooncake
├── mooncake-common ────────────【基础公共层，被所有子系统依赖】
├── mooncake-transfer-engine ───【核心数据传输引擎，全栈底座】
│   └── tent/ ─────────────────── 下一代 TENT：声明式 slice spraying
├── mooncake-store ─────────────【分布式 KVCache 存储引擎，最大子系统】
├── mooncake-p2p-store ────────【P2P 存储 / checkpoint-engine 原型】
├── mooncake-ep ────────────────【专家并行 Expert Parallelism】
├── mooncake-pg ────────────────【进程组 Process Group 后端】
├── mooncake-rl ────────────────【RL 场景示例层】
├── mooncake-integration ──────【Python 绑定与集成层】
└── mooncake-wheel ────────────【pip 打包分发层】
```

对应顶层构建入口见 `CMakeLists.txt:71-195`，各子系统通过 `add_subdirectory` 接入，并通过一系列 `WITH_*` 开关控制构建（`WITH_TE` / `WITH_STORE` / `WITH_EP` / `WITH_P2P_STORE` 等）。

## 各子系统内部结构

### mooncake-common（基础公共层）

```
mooncake-common/
├── include/        default_config.h · environ.h · duration_utils.h
├── src/ · tests/
├── cmake 模块      FindJsonCpp · FindGLOG · FindMpi · FindUrma · SetupPython · SetupPyTorchEnv · common.cmake
├── etcd/           etcd 客户端封装 ──┐
└── k8s-lease/      K8s 租约 leader election ─┤ 供 Store HA 复用
```

详见 [`../01-common/`](../01-common/)。

### mooncake-transfer-engine（传输引擎 TE）

```
mooncake-transfer-engine/
├── 引擎核心        transfer_engine(_impl/_c) · multi_transport · topology
│                   transfer_metadata(+plugin) · config · graceful_shutdown · memory_location · show_links
├── transport/      传输后端插件族（横向跨协议/硬件）
│                   rdma · tcp · nvlink · intranode_nvlink · efa · nvmeof · cxl · barex · cxi · rpc_communicator
│                   异构加速器: ascend · hip · maca · sunrise_link · kunpeng
├── 内存分配器      ascend_allocator · sunrise_allocator · ub_allocator · cuda_alike · hip_device_guard · gpu_vendor
├── tent/           下一代 TENT（slice spraying / failover / qos / selector / tebench）
├── rust/ · benchmark/ · example/ · scripts/ · tests/
```

TE 为全栈底座：Store 的 `transfer_task`、P2P、EP/PG 的底层搬移均走 TE API。详见 [`../02-transfer-engine/`](../02-transfer-engine/)。

### mooncake-store（分布式存储引擎，最大子系统）

```
mooncake-store/
├── 控制面 Master   master(+service/admin) · master_client · master_metric_manager
│                   master_snapshot(_manager/_repository) · tenant_quota(+policy_store) · ha_metric_manager
├── 数据面 Client   real_client · client_service · client_buffer · aligned_client_buffer
│                   dummy_client · client_metric · pyclient · store_c
├── Segment/对象    segment · storage_backend · offset_allocator · mmap_arena · memory_alloc · allocator
├── 文件/块后端     file_storage · file_interface · posix_file · uring_file
├── 多级存储后端    storage/distributed · device/(accelerator: cuda_like/hip/ascend/sunrise/runtime+registry)
│                   hf3fs(3FS) · spdk(NVMe) · cachelib_memory_allocator
├── 复制与任务      replica(+selection) · transfer_task · task_manager · thread_pool · deadline_scheduler
├── 缓存/驱逐       local_hot_cache · eviction_strategy · count_min_sketch · hybrid_metric
├── HA 高可用       ha/(common/leadership/oplog/snapshot/standby_controller) · hot_standby_service · standby_state_machine
├── Engram 表存储   engram/engram_store ──embedding 表后端（与 Conductor 独立，见 03-store/engram-conductor.md）
├── KV 事件         kv_event/kv_event_publisher
├── 序列化/元数据   serialize/ · etcd_helper · k8s_lease_helper · http_metadata_server · metadata_store
├── RPC/工具        rpc_service(+helper+types) · uds_transport · config_helper · master_config · utils · version
└── go/ · rust/ · conf/ · benchmarks · tests
```

详见 [`../03-store/`](../03-store/)。

### mooncake-p2p-store（P2P 存储）

去中心化 peer-to-peer 存储；生产版独立为 [checkpoint-engine](https://github.com/MoonshotAI/checkpoint-engine)，在 K1.5/K2 训练中 ~20s 更新 1T 参数。详见 [`../04-p2p-store/`](../04-p2p-store/)。

### mooncake-ep / mooncake-pg（弹性 MoE 执行层）

- **EP**：DeepEP 兼容低延迟 dispatch/combine + `active_ranks` 容错；PyTorch CUDA 扩展。见 [`../05-ep/`](../05-ep/)。
- **PG**：`torch.distributed` 后端，提供 collective + rank 故障检测 / 恢复；依赖 EP。见 [`../06-pg/`](../06-pg/)。

### mooncake-rl（RL 示例）

基于 TE/Store 的权重与隐状态传输示例（`examples/rl_samples.py`）。见 [`../07-rl/`](../07-rl/)。

### mooncake-integration（Python 绑定层）

```
mooncake-integration/
├── transfer_engine/transfer_engine_py.cpp   TE Python 扩展
├── store/store_py.cpp + async_store.py + engram_store_py.cpp
│            + buffer_pool.cpp + parallel_read/write.h        Store Python 扩展
└── allocator.py / allocator_ascend_npu.py / fabric_allocator_utils.py / integration_utils.h
```

是 C++ 内核与上层 Python 包之间的桥。见 [`../08-integration/`](../08-integration/)。

### mooncake-wheel（打包分发）

`mooncake/` Python 包：CLI 工具、buffer_pool、store_service、connector、EP/PG 绑定、ssd 管理、proxy server 等，最终发布为 `mooncake-transfer-engine` 轮子。见 [`../09-wheel/`](../09-wheel/)。

## 横向支撑（非 CMake 子系统，但属架构一部分）

- `docs/` — Sphinx 官方文档站（用户向）。
- `monitoring/` — Grafana + Prometheus 监控栈。
- `benchmarks/` · `scripts/`(CI / tone_tests) · `docker/` · `extern/`(pybind11…)。
- `FAST25-release/` — 论文与 traces。

详见 [`../10-cross-cutting/`](../10-cross-cutting/) 与 [`../11-ecosystem/`](../11-ecosystem/)。

## Tensor-Centric 全栈定位

```
        ┌─────────────────────────────────────────────┐
        │        外部生态 (SGLang / vLLM / TRT-LLM…)   │  11-ecosystem
        ├─────────────────────────────────────────────┤
        │  mooncake-wheel    pip 分发 + CLI + 集成连接器  │  09-wheel
        ├─────────────────────────────────────────────┤
        │  mooncake-integration  Python C++ 扩展桥       │  08-integration
        ├──────────────────┬──────────────────────────┤
        │  mooncake-ep/pg  │  mooncake-store           │  05/06 · 03-store
        │  弹性 MoE 执行    │  分布式 KVCache/权重存储    │
        ├──────────────────┴──────────────────────────┤
        │  mooncake-transfer-engine (+tent)            │  02-transfer-engine
        │  异构传输 + 内存分配 + 拓扑/元数据              │
        ├─────────────────────────────────────────────┤
        │  mooncake-common  cmake / etcd / k8s-lease   │  01-common
        └─────────────────────────────────────────────┘
                       ▲ Tensor 数据载体贯穿全栈 ▲
```

每一层只依赖其下层，但**异构硬件后端**与**元数据/HA**会在多个层横向复用，详见 [`dependency-graph.md`](dependency-graph.md)。

## 参考文档

- 官方架构概览：`../../docs/source/design/architecture.md`
- 主仓库 README：`../../README.md`
- FAST'25 论文：<https://www.usenix.org/system/files/fast25-qin.pdf>
