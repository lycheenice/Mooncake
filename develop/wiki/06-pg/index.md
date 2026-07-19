# mooncake-pg（Process Group 进程组后端）

## 1. 定位

`mooncake-pg` 是 Mooncake 的 **`torch.distributed` ProcessGroup 后端**（Python 模块 `mooncake.pg`），把 Mooncake 的传输与失效检测能力以 PyTorch 标准 collective API 暴露出来，并在此基础上提供 **rank 故障检测 / 上报与弹性 rank 恢复**，使推理服务在 partial rank failure 下不必整体重启即可继续服务。

导入 `mooncake.pg` 时向 PyTorch 注册两个后端（`pg_py.cpp:30-44`）：

- `mooncake` —— accelerator（CUDA / MUSA，按构建决定）backend；
- `mooncake-cpu` —— CPU backend。

`MooncakeBackend` 继承 `c10d::ProcessGroup`（`mooncake_backend.h:59`），用户像使用 NCCL/Gloo 一样调用 `dist.init_process_group()` / `dist.all_reduce()` / `dist.new_group()` / `dist.batch_isend_irecv()`。Mooncake PG 的独特价值在于：

- **active-rank 跟踪**：通过 `MooncakeBackendOptions(activeRanks)` 注入 `int32` 健康张量，collective 跳过 inactive rank，不阻塞等待。
- **弹性恢复协议**：`extend_group_size_to` / `get_peer_state` / `recover_ranks` / `join_group` 等 primitive 让替换 rank 发布 metadata、加入既有 group 并被健康 rank 激活。
- **底座复用**：默认自建 `TransferEngine`；高级集成可经 `pg.set_transfer_engine(engine)` 注入外部 TE（`mooncake_backend.h:167`），与 EP / Store 共享同一引擎。

服务对象广泛：[论文](https://arxiv.org/abs/2605.10670) "Surviving Partial Rank Failures in Wide Expert-Parallel MoE Inference" 描述了 PG + EP 的失效容忍 MoE 推理设计；SGLang 的 Elastic EP 即基于该后端。

## 2. 目录 / 源码结构

```
mooncake-pg/
├── CMakeLists.txt              IDE 模式(EPP_USE_IDE); 生产模式由顶层 cmake -P 驱动
├── BuildPgExt.cmake            生产构建脚本: 跑 setup.py + stage .so
├── setup.py                    torch CUDAExtension, 链接 -l:engine.so
├── include/
│   ├── mooncake_backend.h      MooncakeBackend / MooncakeP2PShim / MooncakeBackendOptions
│   ├── mooncake_worker.cuh · mooncake_worker_kernels.cuh   通信 worker (collective 内核)
│   ├── connection_poller.h     ConnectionContext: peer 连接状态轮询
│   ├── p2p_proxy.h             P2PProxy: P2P send/recv 代理
│   └── pg_utils.h
├── src/
│   ├── pg_py.cpp               pybind11 绑定 + 注册 mooncake/mooncake-cpu
│   ├── mooncake_backend.cpp    ProcessGroup collective 实现
│   ├── mooncake_worker.cu · mooncake_worker_host.cpp · mooncake_worker_thread.cpp   CUDA worker
│   ├── connection_poller.cpp   peer-state polling (恢复探测)
│   └── p2p_proxy.cpp           P2P send/recv 实现
├── tests/                      test_pg_collectives/elastic/init_functional/p2p/inference_* + pg_test_utils
└── benchmark/                  pgbench.py · pgbench_utils.py · p2p_regular_k_bench.py · README.md
```

### 构建开关与流程

PG 没有 `WITH_PG` 独立开关，**挂在 `WITH_EP` 之下**（顶层 `CMakeLists.txt:96`）：开启 `WITH_EP` 即同时构建 EP 与 PG。生产路径（`EP_USE_IDE=OFF`）下定义 custom target `mooncake_pg_ext`（`CMakeLists.txt:158`），`DEPENDS engine mooncake_ep_ext`（`CMakeLists.txt:173`），以 `cmake -P BuildPgExt.cmake` 驱动，流程与 EP 同构：

1. `BuildPgExt.cmake:65` —— 在 `mooncake-wheel/mooncake/` 建 `engine.so` 符号链接。
2. `BuildPgExt.cmake:77` —— 对每个 `EP_TORCH_VERSIONS` 跑 `python setup.py build_ext`，产出 `mooncake.pg_<torch_ver>`（`setup.py:27`）。
3. `BuildPgExt.cmake:103` —— 把 `.so` 复制到 `EP_PG_STAGING_DIR`。

`setup.py:103` 的 `CUDAExtension` sources 包括 `pg_py.cpp`、`mooncake_backend.cpp`、`p2p_proxy.cpp`、`mooncake_worker.cu` + host + thread、`connection_poller.cpp`；链接 `-l:engine.so`（`setup.py:122`，即 TE 的 Python 扩展），并支持 CUDA / MUSA / MACA。

IDE 模式下 `mooncake-pg/CMakeLists.txt` + `src/CMakeLists.txt:1` 编译 `mooncake_pg` 库，链接 `${TORCH_LIBRARIES} transfer_engine ibverbs mlx5`。

> 依赖关系存在两个方向：**构建序** PG 依赖 EP（`mooncake_pg_ext DEPENDS mooncake_ep_ext`，先把 EP `.so` 入 staging）；**运行时** EP 依赖 PG（EP `Buffer` 由 Mooncake Backend 进程组构造，用其 `all_gather`/`all_to_all` 交换 bootstrap metadata，并读取其 active-rank 状态）。详见 [`../05-ep/index.md`](../05-ep/index.md)。

## 3. 核心概念

### `MooncakeBackend` 与 `MooncakeBackendOptions`

`MooncakeBackend`（`mooncake_backend.h:59`）继承 `c10d::ProcessGroup`，实现 `allreduce` / `broadcast` / `allgather` / `_allgather_base` / `_reduce_scatter_base` / `alltoall` / `barrier` / `reduce` / `gather` / `scatter` / 单 tensor `send`·`recv`（`mooncake_backend.h:105-152`）。`getSize()` 返回 `meta_->activeSize`（`mooncake_backend.h:101`），即 PyTorch `dist.get_world_size()` 看到的"活动规模"，可与保留容量 `size` 不同。

`MooncakeBackendOptions`（`mooncake_backend.h:61`）字段：

| 字段 | 含义 |
|------|------|
| `activeRanks_` | `torch.int32` 健康张量；`mooncake` 放 accelerator，`mooncake-cpu` 放 CPU |
| `isExtension_` | 该进程是否为加入/替换 rank |
| `maxWorldSize_` | 预留容量上限；>0 时 `activeRanks` 需按此长度分配，保留 inactive 槽位 |

### `MooncakeP2PShim`（P2P 桥）

PyTorch 的 P2P 派发（`batch_isend_irecv` / `isend` / `irecv`）需要 `c10d::Backend` 实例。`MooncakeBackend` 继承的是 `ProcessGroup` 而非 `Backend`，故用轻量 `MooncakeP2PShim`（`mooncake_backend.h:33`）注册进 `deviceTypeToBackend_` map，仅 delegate `send` / `recv` / `barrier` / `getBackendName` / `supportsCoalescing` 给 owner `MooncakeBackend`。

### active ranks 与动态 world size

PG 区分两个概念：

- **保留容量 `size`**：backend 知道的 rank 槽位总数。
- **活动规模 `activeSize`**：PyTorch API 可见的当前 group 大小。

`max_world_size > world_size` 时，PG 预留额外槽位但标记 inactive；健康 rank 据此 poll joiner metadata 并在之后激活，无需重建进程组。`extend_group_size_to(size)` 可动态扩容；新扩展的 rank 默认 inactive，须经 `get_peer_state()` + `recover_ranks()` 才参与 collective。

### 弹性恢复两阶段协议

```
Healthy ranks          Joining rank           Store / metadata
init(W, max_world_size=N)                     init(N, is_extension=True)
                                              publish local peer metadata
get_peer_state(join_ranks) ───────────────►
recover_ranks(join_ranks) ─────────────────►  extension state
                                              join_group() returns
collectives include recovered ranks ◄────────
```

健康 rank 职责：预留容量 → 按一致顺序 `get_peer_state` → `recover_ranks` → 刷新上层（如 EP buffer 的 `update_ep_member`）。加入 rank 职责：`is_extension=True` 初始化 → 发布 peer metadata → `join_group()` 阻塞至健康 rank 发布 extension state → 返回后正常参与 collective。Python 入口见 `pg_py.cpp:122-127`：`get_active_ranks` / `get_num_synced_ranks` / `extend_group_size_to` / `get_peer_state` / `recover_ranks` / `join_group`。

### Transfer Engine 所有权与拓扑

默认 PG 自建 `TransferEngine`（静态 `engine_`）；可经 `pg.set_transfer_engine(engine)`（`pg_py.cpp:115` → `MooncakeBackend::setExternalEngine`）注入外部引擎，调用方负责其生命周期。`getPreferredHca(location)`（`mooncake_backend.h:169`）借助 TE 拓扑矩阵选择 NUMA 亲和 HCA，优化多 NIC 带宽聚合。

### 故障 / 恢复边界

PG 只提供**通信层**的恢复 primitive；策略决策（哪些 rank 可替换、何时停路由、如何在替换进程重建模型状态、何时刷新 EP/scheduler）由上层系统负责。`recover_ranks()` 仅激活通信路径，不重建模型权重 / KV cache / 路由策略。

## 4. 交叉引用

- 上层 EP：[`../05-ep/index.md`](../05-ep/index.md) —— EP `Buffer` 从 Mooncake Backend 进程组构造，复用其 collective 做 bootstrap 并消费其 active-rank 状态；PG 恢复后 EP 须调 `update_ep_member()` 刷新 peer metadata。
- 底层搬移：[`../02-transfer-engine/index.md`](../02-transfer-engine/index.md) —— `engine.so` 为 TE Python 扩展，PG 默认自建或经 `set_transfer_engine` 注入；`getPreferredHca` 复用 TE 拓扑。
- 官方设计：[`../../docs/source/design/mooncake-backend-pg.md`](../../docs/source/design/mooncake-backend-pg.md)。
- Python API 参考（含 PG quick start）：[`../../docs/source/python-api-reference/ep-backend.html`](../../docs/source/python-api-reference/ep-backend.html)。
- 故障排查：[`../../docs/source/troubleshooting/pg-ep-troubleshooting.md`](../../docs/source/troubleshooting/pg-ep-troubleshooting.md)。
- 论文：[Surviving Partial Rank Failures in Wide Expert-Parallel MoE Inference](https://arxiv.org/abs/2605.10670)。

## 5. 关键源码入口

- `CMakeLists.txt:158` —— `mooncake_pg_ext` custom target（构建序在 EP 之后）。
- `CMakeLists.txt:173` —— `DEPENDS engine mooncake_ep_ext`。
- `mooncake-pg/BuildPgExt.cmake:65` —— `[PG]` `engine.so` 符号链接。
- `mooncake-pg/BuildPgExt.cmake:77` —— `setup.py build_ext` 实际编译。
- `mooncake-pg/BuildPgExt.cmake:103` —— stage `.so` → `EP_PG_STAGING_DIR`。
- `mooncake-pg/setup.py:27` —— `mooncake.pg_<torch_ver>` 模块名。
- `mooncake-pg/setup.py:103` —— `CUDAExtension(...)`（sources / `-l:engine.so` 见 `:122`）。
- `mooncake-pg/src/CMakeLists.txt:1` —— IDE 模式 `mooncake_pg` 库（`target_link_libraries ... transfer_engine ibverbs mlx5`）。
- `mooncake-pg/src/pg_py.cpp:30` —— `register_backend("mooncake-cpu"/"mooncake", ...)`。
- `mooncake-pg/src/pg_py.cpp:110` —— `PYBIND11_MODULE` 入口。
- `mooncake-pg/src/pg_py.cpp:115` —— `set_transfer_engine`（注入外部 TE）。
- `mooncake-pg/src/pg_py.cpp:122` —— `get_active_ranks` / `get_num_synced_ranks`。
- `mooncake-pg/src/pg_py.cpp:124` —— `extend_group_size_to` / `get_peer_state` / `recover_ranks` / `join_group`。
- `mooncake-pg/include/mooncake_backend.h:33` —— `MooncakeP2PShim`（P2P 桥）。
- `mooncake-pg/include/mooncake_backend.h:59` —— `MooncakeBackend : c10d::ProcessGroup`。
- `mooncake-pg/include/mooncake_backend.h:61` —— `MooncakeBackendOptions`（`activeRanks_` / `isExtension_` / `maxWorldSize_`）。
- `mooncake-pg/include/mooncake_backend.h:101` —— `getSize()` 返回 `activeSize`。
- `mooncake-pg/include/mooncake_backend.h:115` —— `allreduce` / `allgather`（`:119`）/ `alltoall`（`:132`）。
- `mooncake-pg/include/mooncake_backend.h:167` —— `setExternalEngine`（外部 TE 注入）。
- `mooncake-pg/include/mooncake_backend.h:205` —— `extendGroupSizeTo` / `getPeerState`（`:207`）/ `recoverRanks`（`:209`）/ `joinGroup`（`:211`）。
- `mooncake-pg/src/mooncake_backend.cpp` —— ProcessGroup collective 实现。
- `mooncake-pg/src/mooncake_worker.cu` —— collective CUDA worker kernel。
- `mooncake-pg/src/connection_poller.cpp` —— `ConnectionContext` peer-state 轮询（恢复探测）。
- `mooncake-pg/src/p2p_proxy.cpp` —— `P2PProxy` send/recv 实现。
- `mooncake-pg/tests/test_pg_elastic.py` —— 弹性恢复（`extend_group_size_to` / `get_peer_state` / `recover_ranks` / `join_group`）测试。
- `mooncake-pg/tests/test_pg_collectives.py` · `test_pg_p2p.py` · `test_pg_init_functional.py` —— collective / P2P / 初始化功能测试。
- `mooncake-pg/benchmark/pgbench.py` —— collective 性能基准。

## 参考文档

- 官方设计：[`../../docs/source/design/mooncake-backend-pg.md`](../../docs/source/design/mooncake-backend-pg.md)
- Python API 参考：[`../../docs/source/python-api-reference/ep-backend.html`](../../docs/source/python-api-reference/ep-backend.html)
- PG/EP 故障排查：[`../../docs/source/troubleshooting/pg-ep-troubleshooting.md`](../../docs/source/troubleshooting/pg-ep-troubleshooting.md)
- 论文：[Surviving Partial Rank Failures in Wide Expert-Parallel MoE Inference](https://arxiv.org/abs/2605.10670)
