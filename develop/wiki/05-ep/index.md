# mooncake-ep（Expert Parallelism 专家并行）

## 1. 定位

`mooncake-ep` 是 Mooncake 的 **MoE 专家并行通信运行时**，为大规模 MoE 推理提供低延迟的 token dispatch / combine 内核。它遵循 [DeepEP](https://github.com/deepseek-ai/DeepEP) 的低延迟编程模型，并在其之上加入两点关键扩展：

- **Mooncake 传输集成**：inter-node 走 IBGDA（GPU Direct RDMA），intra-node 走 NVLink P2P/IPC，必要时退回 PyTorch collective 的 Python fallback。
- **Rank activeness 感知**：dispatch / combine 接受 `active_ranks: [num_ranks] int32` 健康张量，内核以 `timeout_us` 轮询接收信号，对超时无进展的源 rank 标记为失活并跳过，使 MoE 推理在 **partial rank failure** 下继续服务而非整体崩溃。

EP 以 **PyTorch CUDA Extension** 形式构建（`torch.utils.cpp_extension.BuildExtension`），import 时按 `torch.__version__` 加载 version-suffixed 模块（`mooncake.ep_<torch_ver>`）。它从 `torch.distributed` 进程组（通常即 [`mooncake-pg`](../06-pg/index.md) 的 Mooncake Backend）构造，借助 group 的 `all_gather` / `all_to_all` 交换 RDMA MR / QPN / GID·LID / IPC handle 等 bootstrap 元数据。

生产集成：SGLang 的 **Elastic Expert Parallel**（[blog](https://lmsys.org/blog/2026-03-25-eep-partial-failure-tolerance/)）以 Mooncake collective + EP 内核作为后端（`--elastic-ep-backend mooncake`），实现大规模 MoE 的部分失效容忍服务。

## 2. 目录 / 源码结构

```
mooncake-ep/
├── CMakeLists.txt              IDE 模式构建(EPP_USE_IDE); 生产模式由顶层 cmake -P 驱动
├── BuildEpExt.cmake            生产构建脚本: 跑 setup.py + 把 .so 暂存到 staging dir
├── setup.py                    torch CUDAExtension, 链接 -l:engine.so
├── include/
│   ├── mooncake_ep_buffer.h             MooncakeEpBuffer / BufferPair / BufferLayout
│   ├── mooncake_ep_api.cuh · configs · device · event · launch · utils · exception
│   └── elastic/                          弹性(高阶)层: topology/hybrid/PTX/dispatch·combine official 内核
│       ├── mooncake_ep_elastic_buffer.h  MooncakeElasticBuffer (ElasticConfig/ElasticTopology)
│       ├── mooncake_ep_elastic_dispatch_official.cuh / combine_official.cuh / hybrid_*.cuh
│       └── …(layout · math · ptx · transport · comm · launch · compiled · epilogue)
├── src/
│   ├── ep_py.cpp               pybind11 绑定: Buffer / ElasticBuffer / EventHandle
│   ├── mooncake_ep_buffer.cpp  MooncakeEpBuffer 实现
│   ├── mooncake_ep_elastic_buffer.cpp   ElasticBuffer 实现 + topology 发现
│   ├── mooncake_ep_kernel.cu · mooncake_ep_elastic_kernel.cu   dispatch/combine CUDA 内核
├── tests/                      test_ep_grid.py · test_elastic_buffer.py
├── benchmarks/                 elastic_buffer_perf.py
└── .gitignore
```

### 构建开关与流程（顶层 `CMakeLists.txt`）

| 开关 | 位置 | 说明 |
|------|------|------|
| `WITH_EP` | `CMakeLists.txt:21` | EP/PG 总开关，默认 OFF |
| `EP_USE_IDE` | `CMakeLists.txt:95` | 仅 IDE 索引；开启时直接 `add_subdirectory(mooncake-ep)`，**勿用于生产** (`CMakeLists.txt:98`) |
| `EP_TORCH_VERSIONS` | `CMakeLists.txt:113` | 分号分隔的 PyTorch 版本列表（空=用当前安装的 torch） |
| `TORCH_CUDA_ARCH_LIST` | `CMakeLists.txt:120` | 转发给 torch 扩展构建，默认 `8.0;9.0` |
| `EP_PG_STAGING_DIR` | `CMakeLists.txt:132` | EP/PG .so 暂存目录，wheel 打包后注入，避免 auditwheel/patchelf 碰 CUDA fatbin |

生产路径（`EP_USE_IDE=OFF`）下，顶层定义 custom target `mooncake_ep_ext`（`CMakeLists.txt:139`，`DEPENDS engine` 即 `CMakeLists.txt:154`），以 `cmake -P BuildEpExt.cmake` 驱动：

1. `BuildEpExt.cmake:61` —— 在 `mooncake-wheel/mooncake/` 建 `engine.so` 符号链接指向已构建的 TE Python 扩展，供 setup.py `-l:engine.so` 链接。
2. `BuildEpExt.cmake:76` —— 对每个 `EP_TORCH_VERSIONS`（或当前 torch）跑 `python setup.py build_ext --build-lib .`；`setup.py:31` 由此产出 `mooncake.ep_<torch_ver>`。
3. `BuildEpExt.cmake:102` —— 把 `.so` 复制到 `EP_PG_STAGING_DIR`，与 wheelpack 解耦，规避 `cudaErrorInvalidKernelImage`（fatbin 被 patchelf 改写）。
4. `mooncake_pg_ext` 依赖 `engine mooncake_ep_ext`（`CMakeLists.txt:173`），EP 先于 PG。

`setup.py` 同时支持 CUDA / MUSA（`MOONCAKE_EP_USE_MUSA`，`setup.py:7`）/ MACA（`MOONCAKE_EP_USE_MACA`，`setup.py:8`）三类设备编译。IDE 模式下 `src/CMakeLists.txt:1` 编译 `mooncake_ep` 静态库，链接 `${TORCH_LIBRARIES} transfer_engine ibverbs mlx5`。

## 3. 核心概念

### 两层抽象：`Buffer` 与 `ElasticBuffer`

- **`MooncakeEpBuffer`**（`mooncake_ep_buffer.h:66`，Python 类名 `Buffer`）：底层 runtime，持有 rank/world、GDR workspace、`P2pTransport`（NVLink）/ `RdmaTransport`（IBGDA）双设备传输、通信 stream 与 workspace。构造时可传入 `TransferEngine*` 复用其 device transports，否则自建自持。
- **`MooncakeElasticBuffer`**（`elastic/mooncake_ep_elastic_buffer.h:76`，Python 类名 `ElasticBuffer`）：高阶封装，自动发现 `ElasticTopology`（scale-up/scale-out、NVLink/RDMA rank 数、是否启用 hybrid），提供 `ElasticConfig`（`use_fp8_dispatch` / `deterministic` / `allow_hybrid_mode` / `prefer_overlap_with_compute` / `num_cpu_timeout_secs` 等）与 `calculate_buffer_size` 容量推算。两者通过 `native_buffer()` 桥接。

### Dispatch / Combine 三阶段数据流

1. **Dispatch**：每 rank 把本地 token hidden states `x [num_tokens, hidden]` 按 `topk_idx` 路由到专家所在 rank，接收方按本地专家 packing。
2. **Expert compute**：在 packed 输入上跑本地专家。
3. **Combine**：把专家输出连同 `topk_weights` 路由回原 token owner 并 reduce；`combine()` 消费 `dispatch()` 返回的 `src_info` / `layout_range` handle。支持 `zero_copy` 模式：先 `get_next_combine_buffer(handle)` 写入再 `combine(..., zero_copy=True)`。

双缓冲 `BufferPair`（`mooncake_ep_buffer.h:30`）为每对收发分配 4 块 RDMA buffer（send/recv signal + send/recv data）；容量由 `get_ep_buffer_size_hint(num_max_dispatch_tokens_per_rank, hidden, num_ranks, num_experts)` 推算（`mooncake_ep_buffer.h:197`），需按峰值 token 数而非均值预留。

### 三种执行路径

| 模式 | 用途 | 触发条件 |
|------|------|---------|
| IBGDA / RDMA fast path | inter-node GPU 显存直传 | RDMA 可用且 metadata/QP 交换成功 |
| P2P / IPC fast path | intra-node NVLink | peer 可达且 IPC handle 交换成功 |
| Python fallback | 功能兜底 | 上述均不可用（性能降级，仅供正确性/受限测试） |

`use_fast_path()`（`mooncake_ep_buffer.h:143`）依据 `ibgda_disabled_` 与 `P2pTransport::allPeersAccessible()` 判定。

### active_ranks 与失效容忍

dispatch / combine 都接收 `active_ranks`（rank 级，`[num_ranks] int32`）。内核轮询各源 rank 的接收信号；若 `timeout_us != -1` 且某源 rank 在超时前无进展，则置 `active_ranks[src_rank] = 0` 并跳过。该张量与 PG backend 级 `MooncakeBackendOptions(active_ranks)` 相关但非自动相同，集成方需在 scheduler / PG 状态 / EP buffer 之间一致传播健康状态。

### 恢复集成

rank 失效后的协同流程：

1. PG active-rank 状态先停止 collective 等待失效 rank；
2. scheduler / MoE routing 停止把 token 路由到不可用专家；
3. 替补 rank 经 PG 弹性协议加入；
4. EP buffer 调 `Buffer.update_ep_member()`（见 [`../../docs/source/python-api-reference/ep-backend.html`](../../docs/source/python-api-reference/ep-backend.html)）刷新 peer metadata / QP，再让恢复 rank 承担 dispatch/combine。

### Stream 同步模型

EP 返回 `EventHandle`（`ep_py.cpp:20`）与可选 hook：`return_recv_hook=False` 时调 `event.current_stream_wait()`；`True` 时在选定 overlap 点调 `hook()`；`async_finish=True` 额外记 tensor 到 event helper 以保 lifetime，适配 CUDA graph。

## 4. 交叉引用

- 上层依赖：[`../06-pg/index.md`](../06-pg/index.md) —— EP 由 Mooncake Backend 进程组构造，PG 先于 EP 构建（`CMakeLists.txt:173`）；恢复流程由 PG 弹性协议驱动。
- 底层搬移：[`../02-transfer-engine/index.md`](../02-transfer-engine/index.md) —— `engine.so` 即 TE 的 Python 扩展；EP 可复用 TE 拥有的 `P2pTransport` / `RdmaTransport`（`mooncake_ep_buffer.h:108`），否则自建。
- 官方设计：[`../../docs/source/design/mooncake-ep.md`](../../docs/source/design/mooncake-ep.md)。
- 官方 API 参考：[`../../docs/source/python-api-reference/ep-backend.html`](../../docs/source/python-api-reference/ep-backend.html)（含 `Buffer.update_ep_member()` 等）。
- 故障排查：[`../../docs/source/troubleshooting/pg-ep-troubleshooting.md`](../../docs/source/troubleshooting/pg-ep-troubleshooting.md)（PyTorch 版本不匹配、`activeRanks` 类型/设备错配等）。
- SGLang 弹性 EP 集成：[Elastic EP blog](https://lmsys.org/blog/2026-03-25-eep-partial-failure-tolerance/)、`--elastic-ep-backend mooncake`（`../../docs/source/getting_started/examples/sglang-integration-v1.md`）。

## 5. 关键源码入口

- `CMakeLists.txt:21` —— `option(WITH_EP ...)` 总开关。
- `CMakeLists.txt:95` —— `EP_USE_IDE`（IDE 索引专用）。
- `CMakeLists.txt:113` —— `EP_TORCH_VERSIONS` 解析（env / cache）。
- `CMakeLists.txt:120` —— `TORCH_CUDA_ARCH_LIST` 默认 `8.0;9.0`。
- `CMakeLists.txt:139` —— `mooncake_ep_ext` custom target（`DEPENDS engine` 见 `:154`）。
- `mooncake-ep/BuildEpExt.cmake:61` —— `engine.so` 符号链接（解 `-l:engine.so` 链接）。
- `mooncake-ep/BuildEpExt.cmake:76` —— `setup.py build_ext` 实际编译。
- `mooncake-ep/BuildEpExt.cmake:102` —— stage `.so` → `EP_PG_STAGING_DIR`。
- `mooncake-ep/setup.py:31` —— version-suffixed 模块名 `mooncake.ep_<torch_ver>`。
- `mooncake-ep/setup.py:113` —— `CUDAExtension(...)` 声明（sources/include_dirs/`-l:engine.so` 见 `:130`）。
- `mooncake-ep/src/CMakeLists.txt:1` —— IDE 模式 `mooncake_ep` 库（`DEPENDS transfer_engine ibverbs mlx5`）。
- `mooncake-ep/src/ep_py.cpp:15` —— `PYBIND11_MODULE` 入口。
- `mooncake-ep/src/ep_py.cpp:70` —— `Buffer`（`MooncakeEpBuffer`）绑定。
- `mooncake-ep/src/ep_py.cpp:89` —— `ElasticBuffer`（`MooncakeElasticBuffer`）绑定。
- `mooncake-ep/include/mooncake_ep_buffer.h:66` —— `MooncakeEpBuffer` 结构（双 transport + workspace）。
- `mooncake-ep/include/mooncake_ep_buffer.h:116` —— `dispatch(...)` 签名（含 `active_ranks`、`timeout_us`）。
- `mooncake-ep/include/mooncake_ep_buffer.h:124` —— `combine(...)` 签名（`zero_copy` / `out`）。
- `mooncake-ep/include/mooncake_ep_buffer.h:143` —— `use_fast_path()`：IBGDA 或 P2P 全可达。
- `mooncake-ep/include/mooncake_ep_buffer.h:197` —— `get_ep_buffer_size_hint`。
- `mooncake-ep/include/elastic/mooncake_ep_elastic_buffer.h:76` —— `MooncakeElasticBuffer`。
- `mooncake-ep/include/elastic/mooncake_ep_elastic_buffer.h:89` —— `calculate_buffer_size`。
- `mooncake-ep/include/elastic/mooncake_ep_elastic_buffer.h:100` —— Elastic `dispatch`（含 `cached_handle` overlap）。
- `mooncake-ep/include/elastic/mooncake_ep_elastic_buffer.h:164` —— `discover_topology`（scale-up/scale-out 探测）。
- `mooncake-ep/tests/test_ep_grid.py` —— EP grid 正确性测试。
- `mooncake-ep/tests/test_elastic_buffer.py` —— ElasticBuffer 测试。
- `mooncake-ep/benchmarks/elastic_buffer_perf.py` —— Elastic 性能基准。

## 参考文档

- 官方设计：[`../../docs/source/design/mooncake-ep.md`](../../docs/source/design/mooncake-ep.md)
- Python API 参考：[`../../docs/source/python-api-reference/ep-backend.html`](../../docs/source/python-api-reference/ep-backend.html)
- PG/EP 故障排查：[`../../docs/source/troubleshooting/pg-ep-troubleshooting.md`](../../docs/source/troubleshooting/pg-ep-troubleshooting.md)
- 论文：[Surviving Partial Rank Failures in Wide Expert-Parallel MoE Inference](https://arxiv.org/abs/2605.10670)
