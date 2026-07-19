# TENT —— 下一代传输引擎（Transfer Engine NEXT）

## 定位

TENT（Transfer Engine NEXT）是 `mooncake-transfer-engine/tent/` 子树，定位为经典 TE 的**下一代运行时**，面向异构互连、动态拓扑与部分故障的大规模 AI 集群。它的核心目标是：**让应用只声明"搬什么数据"，由运行时决定"怎么搬"**——把传输选择、切片调度、故障转移都收进 runtime，而非暴露给应用。

TENT 引入三大设计：(1) 动态传输选择（runtime 按可达性/质量选 transport 与路径，必要时构造 host memory 中转的 staged transfer）；(2) 基于 telemetry 的细粒度 slice spraying（大传输切 slice，按观测完成时间/队列深度自适应分配，慢路径自然少分）；(3) 运行时内故障处理（路径失效自动停调度、跨 transport failover、rail 冷却与恢复）。

## TENT 与经典 TE 的关系

**增强而非替换，两者并存。** 在同一 `TransferEngine` 门面（`include/transfer_engine.h:52`）后，用 `use_tent_` 标志加编译期 `USE_TENT` 宏切换：

- 经典路径：`impl_`（`TransferEngineImpl`，`src/transfer_engine_impl.cpp`）+ `MultiTransport` 静态选择。
- TENT 路径：`impl_tent_`（`mooncake::tent::TransferEngine`，`tent/src/transfer_engine.cpp`）+ `TransportSelector` 动态策略。
- `transfer_engine.cpp` 用 `#ifndef USE_TENT … #else …` 在编译期决定门面方法委托给哪一侧（经典入口 `transfer_engine.cpp:102`，TENT 入口 `transfer_engine.cpp:435`）。

应用 API 表面一致（`init / openSegment / registerLocalMemory / allocateBatchID / submitTransfer / getTransferStatus`），切换后端无需改业务代码。经典 TE 仍是稳定主力，TENT 提供更激进的调度与容错能力。

## 目录 / 源码结构

```
mooncake-transfer-engine/tent/
├── include/tent/
│   ├── transfer_engine.h            TENT C ABI + C++ 门面（tent_engine_t / tent_request_t）
│   ├── device_plugin.h              设备插件接口（CUDA/ROCm platform plugin）
│   ├── common/                      config · status · types · qos_metrics · concurrent（queue/lock/TLS/pool）· utils（ip/os/random…）
│   ├── metastore/                   etcd · http · redis（元数据后端）
│   ├── metrics/                     config_loader · tent_metrics（yalantinglibs + Prometheus）
│   ├── platform/                    cpu · cuda · rocm · ascend · sunrise · tpu(+pjrt shim) 异构 platform/allocator/probe
│   ├── rpc/                         rpc.h（控制面 RPC）
│   ├── runtime/                     TENT 运行时核心（见下）
│   ├── transport/                   transport 后端头（rdma/tcp/nvlink/mnnvl/shm/gds/io_uring/tpu/ascend/sunrise/bufio/fault_proxy）
│   └── thirdparty/nlohmann/json.h
├── src/
│   ├── transfer_engine.cpp          TENT 门面实现（委托 TransferEngineImpl）
│   ├── transfer_engine_c.cpp        C ABI 绑定
│   ├── common/ config · ip · qos_metrics · status
│   ├── metastore/ etcd · http · redis
│   ├── metrics/ config_loader · tent_metrics
│   ├── platform/ cpu_allocator · cpu_probe + cuda/rocm/ascend/sunrise/tpu 子目录（allocator/probe/stream_pool/pjrt_shim）
│   ├── python/ pybind.cpp           Python 绑定
│   ├── rpc/ rpc.cpp
│   ├── runtime/                     运行时实现（见下）
│   └── transport/                   后端实现（与 include/tent/transport 对应）
├── plugins/
│   ├── cuda/cuda_plugin.cpp         CUDA platform plugin（动态加载）
│   └── rocm/rocm_plugin.cpp         ROCm platform plugin
├── config/
│   ├── transfer-engine.json         默认配置（metadata/transports/metrics/topology…）
│   └── cluster-topology.json        集群拓扑示例
└── tests/                           单测/集成测/bench（failover/qos/selector/slice/rail/shm/tpu…，含 tebench 相关）
```

`runtime/` 是 TENT 的大脑：

| 文件 | 职责 |
|------|------|
| `transfer_engine_impl.cpp` | `TransferEngineImpl`：batch 生命周期、submit/prepare/commit、failover resubmit、runtime queue 调度、progress worker 协调 |
| `transport_selector.cpp` | 配置驱动的 transport/device 选择策略（policy 规则匹配） |
| `control_plane.cpp` | 控制面（metadata/segment 注册/发现） |
| `segment_manager.cpp` / `segment_registry.cpp` / `segment_tracker.cpp` / `segment.cpp` | 本地/远端 segment 管理 |
| `admission_queue.cpp` | 本地传输准入队列（runtime queue + dispatch window） |
| `qos_contract.cpp` / `receiver_credit.cpp` | QoS 契约 + 接收方信用流控 |
| `memory_prober.cpp` | 内存类型探测（决定 transport 可达性） |
| `platform.cpp` | platform 插件加载（CUDA/ROCm/Ascend/…） |
| `proxy_manager.cpp` | staging proxy（host memory 中转）管理 |
| `progress_worker.cpp` | 事件驱动进度推进线程（可选） |
| `slab.cpp` | slab 分配 |
| `topology.cpp` | TENT 自有拓扑 |
| `transport_loader.cpp` | 按配置加载 transport 后端 |
| `metastore.cpp` | metastore 后端桥 |

`transport/rdma/` 在 TENT 中被重写并细化：`buffers/context/cq/endpoint/endpoint_store/ibv_loader/quota/shared_quota/rail_monitor/workers/slice` + `bw_arbitration/promotion_policy/params`，承载 slice spraying 与 rail failover 的核心逻辑。

## 核心概念

### 1. 声明式 API

应用提交 `Request`（opcode + source + target_id + target_offset + length + priority + transport_hint，见 `include/tent/transfer_engine.h:38` 的 `tent_request`），不指定 transport。runtime 通过 `getTransportType`（`transfer_engine_impl.cpp:973`）/ `resolveTransport`（`:1415`）在运行时按配置 policy 与可达性决定。若直连不可达，runtime 自动构造**staged transfer**（经 host memory 中转，`submitStagingTransfer`，`transfer_engine_impl.h:221`），对应用透明。

### 2. Slice Spraying

大传输切 slice，每个 slice 独立调度。`DeviceSelector` 有两种模式（见 [`../../docs/source/design/tent/slice-spraying.md`](../../docs/source/design/tent/slice-spraying.md)）：

- **Baseline**（`enable_smart_scheduling=false`）：本地 NUMA tier 内 round-robin。
- **Smart**（`enable_smart_scheduling=true`）：EWMA 带宽估计 + NUMA 惩罚 + 动态多路径分配，慢路径少分 slice，避免 head-of-line blocking。

相关实现：`transport/rdma/rail_monitor.cpp`（rail 状态观测）、`quota.cpp` / `shared_quota.cpp`（配额）、`bw_arbitration.h`（带宽仲裁）、`promotion_policy.h`（升级策略）。测试：`tests/bw_arbitration_test.cpp` / `promotion_policy_test.cpp` / `rail_monitor_test.cpp`。

### 3. Failover

两层恢复（见 [`../../docs/source/design/tent/failover.md`](../../docs/source/design/tent/failover.md)）：

- **跨 transport failover**：`TransferEngineImpl` 在 completion 阶段发现 task 失败，`resubmitTransferTask`（`transfer_engine_impl.h:206`）bump priority、选下一个 transport、重投。应用只要还有健康路径且 `max_failover_attempts`（`transfer_engine_impl.h:341`）未耗尽，就看不到 `FAILED`。
- **RDMA rail 恢复**：`RailMonitor` 对持续失败的 (local NIC, remote NIC) rail 做指数冷却，成功或冷却到期后恢复。

配置：`max_failover_attempts` / `enable_auto_failover_on_poll` / `enable_progress_worker`（`transfer-engine.json`）。`progressBatch`（`transfer_engine.h:330`）显式推进状态机并始终允许 failover。测试：`tests/failover_test.cpp` / `engine_failover_e2e_test.cpp` / `rdma_cancel_test.cpp` / `fault_proxy_test.cpp`。

### 4. QoS

三级优先级（`PRIO_HIGH=0 / MEDIUM=1 / LOW=2`），每个 worker 线程维护三条优先级队列，配合全局时间片协调与 NUMA-aware 设备过滤（见 [`../../docs/source/design/tent/qos.md`](../../docs/source/design/tent/qos.md)）。实现：`runtime/qos_contract.cpp`、`common/qos_metrics.cpp`、`transport/rdma/quota.cpp` / `shared_quota.cpp`。测试：`tests/qos_contract_test.cpp`。

### 5. Transport Selector

配置驱动的 policy 匹配（见 [`../../docs/source/design/tent/transport-selector.md`](../../docs/source/design/tent/transport-selector.md)）。`include/tent/runtime/transport_selector.h:1` 给出规则示例：按 `segment_type`（memory/file）+ `intent_type` + `priority` 匹配，命中的 `devices` / `transports` 列表决定选路。无 policy 时回退原始行为（File: GDS→IOURING→RDMA；Memory: buffer_transports 顺序）。实现：`runtime/transport_selector.cpp`。测试：`tests/transport_selector_test.cpp` / `transport_hint_test.cpp` / `intent_type_test.cpp`。

### 6. tebench

`tebench` 是端到端基准工具，同时支持经典 TE 与 TENT 后端，跨 `(block_size, batch_size, concurrency)` 组合评估带宽/延迟（见 [`../../docs/source/design/tent/tebench.md`](../../docs/source/design/tent/tebench.md)）。Target/Initiator 两角色，`-DUSE_TENT=ON` 构建 TENT 模式。相关测试/bench：`tests/deadline_promotion_bench.cpp` / `hip_bandwidth_bench.cpp`。

### 7. Metrics

基于 yalantinglibs，兼容 Prometheus，提供 Counter / Histogram（见 [`../../docs/source/design/tent/metrics.md`](../../docs/source/design/tent/metrics.md)）。编译期开关 `TENT_METRICS_ENABLED`（默认 OFF，零开销），运行期 `TentMetrics::setEnabled(false)`。配置见 `transfer-engine.json` 的 `metrics` 段（http_port/latency_buckets/size_buckets…）。实现：`metrics/tent_metrics.cpp` / `config_loader.cpp`。测试：`tests/metrics_config_loader_test.cpp` / `metrics_http_server_test.cpp` / `tent_metrics_example.cpp`。

### 8. Platform 插件

`platform/` 按厂商提供 allocator + probe + stream pool：`cpu` / `cuda` / `rocm` / `ascend` / `sunrise` / `tpu`（含 PJRT ABI shim）。`plugins/cuda` 与 `plugins/rocm` 是动态加载的 platform plugin（`device_plugin.h`）。这与经典 TE 的 `include/` 分配器（见 [`allocators.md`](allocators.md)）语义对应，但为 TENT 重写、可热加载。

## 交叉引用

- 经典 TE 引擎核心：[`index.md`](index.md)
- 经典 transport 后端（TENT transport 子树是其重构细化）：[`transports.md`](transports.md)
- 经典分配器（TENT platform 层对应物）：[`allocators.md`](allocators.md)
- 官方 TENT 总览：[`../../docs/source/design/tent/overview.md`](../../docs/source/design/tent/overview.md)
- Slice Spraying：[`../../docs/source/design/tent/slice-spraying.md`](../../docs/source/design/tent/slice-spraying.md)
- Failover：[`../../docs/source/design/tent/failover.md`](../../docs/source/design/tent/failover.md)
- QoS：[`../../docs/source/design/tent/qos.md`](../../docs/source/design/tent/qos.md)
- Transport Selector：[`../../docs/source/design/tent/transport-selector.md`](../../docs/source/design/tent/transport-selector.md)
- tebench：[`../../docs/source/design/tent/tebench.md`](../../docs/source/design/tent/tebench.md)
- Metrics：[`../../docs/source/design/tent/metrics.md`](../../docs/source/design/tent/metrics.md)
- C++ API：[`../../docs/source/design/tent/cpp-api.md`](../../docs/source/design/tent/cpp-api.md)
- 架构拓扑中 TENT 定位：[`../00-overview/architecture.md`](../00-overview/architecture.md)
- 依赖图（TENT 是 TE 的下一代演进）：[`../00-overview/dependency-graph.md`](../00-overview/dependency-graph.md) §5

## 关键源码入口

| 关注点 | 入口 |
|--------|------|
| TENT C++ 门面 | `mooncake-transfer-engine/tent/src/transfer_engine.cpp:40`（`TransferEngine::available`）起委托 impl_ |
| TENT 运行时核心 | `mooncake-transfer-engine/tent/src/runtime/transfer_engine_impl.cpp:281`（`construct`） |
| 运行时头 | `mooncake-transfer-engine/tent/include/tent/runtime/transfer_engine_impl.h:69` |
| 运行时 submit | `mooncake-transfer-engine/tent/src/runtime/transfer_engine_impl.cpp:1426`（`prepareSubmit`） / `:1485`（`commitPreparedSubmit`） |
| transport 选择 | `mooncake-transfer-engine/tent/src/runtime/transfer_engine_impl.cpp:973`（`getTransportType`） / `:1415`（`resolveTransport`） |
| transport selector 策略 | `mooncake-transfer-engine/tent/src/runtime/transport_selector.cpp`（头 `include/tent/runtime/transport_selector.h:1`） |
| failover resubmit | `mooncake-transfer-engine/tent/include/tent/runtime/transfer_engine_impl.h:206`（`resubmitTransferTask`） |
| staging 中转 | `mooncake-transfer-engine/tent/include/tent/runtime/transfer_engine_impl.h:221`（`submitStagingTransfer`） |
| progress worker 推进 | `mooncake-transfer-engine/tent/src/runtime/progress_worker.cpp` |
| QoS 契约 | `mooncake-transfer-engine/tent/src/runtime/qos_contract.cpp` |
| 准入队列 | `mooncake-transfer-engine/tent/src/runtime/admission_queue.cpp` |
| RDMA rail monitor | `mooncake-transfer-engine/tent/src/transport/rdma/rail_monitor.cpp` |
| RDMA 配额/bw 仲裁 | `mooncake-transfer-engine/tent/src/transport/rdma/quota.cpp` / `shared_quota.cpp`（头 `include/tent/transport/rdma/bw_arbitration.h`） |
| C ABI | `mooncake-transfer-engine/tent/src/transfer_engine_c.cpp` |
| C ABI 接口定义 | `mooncake-transfer-engine/tent/include/tent/transfer_engine.h:128`（`tent_create_engine`）起 |
| Python 绑定 | `mooncake-transfer-engine/tent/src/python/pybind.cpp` |
| platform plugin 接口 | `mooncake-transfer-engine/tent/include/tent/device_plugin.h` |
| CUDA platform plugin | `mooncake-transfer-engine/tent/plugins/cuda/cuda_plugin.cpp` |
| 默认配置 | `mooncake-transfer-engine/tent/config/transfer-engine.json` |
| 集群拓扑示例 | `mooncake-transfer-engine/tent/config/cluster-topology.json` |
| failover 测试 | `mooncake-transfer-engine/tent/tests/failover_test.cpp` / `engine_failover_e2e_test.cpp` |
| selector 测试 | `mooncake-transfer-engine/tent/tests/transport_selector_test.cpp` |
| metrics 测试 | `mooncake-transfer-engine/tent/tests/metrics_http_server_test.cpp` |
