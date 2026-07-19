# mooncake-rl（RL 场景示例层）

## 1. 定位

`mooncake-rl` 是 Mooncake 面向**大规模分布式强化学习（RL）**的**示例 / 集成样例层**，本身不提供独立运行时或库：它不参与顶层 CMake 构建（无 `CMakeLists.txt`、无 `setup.py`、无 `src/`），仅以 Python 示例展示如何用 Mooncake Store / Transfer Engine 在 **rollout（推理）引擎**与 **training 引擎**之间搬移 rollout 数据、模型权重与隐状态。

`mooncake-rl/examples/rl_samples.py` 是一个 dummy RL 训练循环（标注 "adapted from THUDM/slime"），通过 `MooncakeDistributedStore` 的 `put_tensor` / `get_tensor` 把 rollout samples 写入 Store、由训练端取出训练，演示 **rollout ↔ train 解耦** 的数据流拓扑；权重同步与隐状态传输在示例中为 mock，生产路径由外部生态（SGLang P2P weight transfer、TorchSpec、DSpark 等）承载。

定位上强调：本子系统**当前主要是示例**，生产级 RL 集成由 [`../11-ecosystem/`](../11-ecosystem/index.md) 中的下游项目落实，Mooncake 自身只提供底层 TE zero-copy 传输与 Store 张量对象管理能力。

## 2. 目录 / 源码结构

```
mooncake-rl/
└── examples/
    └── rl_samples.py     dummy RL 训练循环，演示 MooncakeDistributedStore 在 rollout↔train 间传数据
```

无构建产物、无 CMake 目标、无 Python 打包。运行示例前需先启动 Mooncake `master` / `metadata` 服务，并保证 `mooncake.store`（`mooncake-transfer-engine` wheel）已安装。

## 3. 核心概念

### 角色与数据流（rl_samples.py）

示例围绕四类对象组织，对应一个典型 RL rollout-train 循环：

| 角色 | 类 | 职责 | 与 Mooncake 的交互 |
|------|----|------|--------------------|
| Training worker | `TrainActor` | 单 GPU/进程训练单元，对 rollout 数据做 forward/backward | — |
| Training group | `TrainGroup` | 多 actor 编排；建 Store 客户端 | `MooncakeDistributedStore()` + `setup(...,"rdma",NIC,...)`；`get_tensor(rollout_key)` 取 rollout 样本 |
| Rollout engine | `RolloutEngine` | 生成 `(obs, action, reward)` 样本 | — |
| Rollout controller | `RolloutController` | dataset load/save；建 Store 客户端 | `MooncakeDistributedStore()` + RDMA `setup`；`put_tensor`/`get_tensor` |
| Rollout manager | `RolloutManager` | 聚合 rollout 样本写 Store | `put_tensor(key=str(rollout_id), samples)` |

`train()` 主循环（`rl_samples.py:335`）的时序：

1. `init_actors` 同步模型初始化 / checkpoint 加载；
2. `RolloutController.load` 恢复上轮 dataset 状态；
3. `init_weight_update_connections` 建立 trainer↔rollout 权重同步通道；
4. `update_weights` 把训练端权重同步给 rollout 端（示例中 mock）；
5. 每轮：`rollout_manager.generate(rollout_id)` 写 Store → `actor_model.train(id, key)` 从 Store 读出训练 → 可选 `save_model` / `eval`。

### 关键 Mooncake API 用法

- `MooncakeDistributedStore()` 实例化（`rl_samples.py:98`、`:232`）。
- `setup(local_server_name, metadata_url, global_segment_size, buffer_size, protocol, device_name, master_server_name)`（`rl_samples.py:100`）：示例以 `512MB` segment、`128MB` buffer、`"rdma"` 协议、`erdma_1`/`mlx5_1` 等 NIC 初始化。
- `put_tensor(key, samples)` / `get_tensor(key)`（`:306` / `:142` / `:315`）：以 rollout id 为 key，把 Python 对象序列化进 Store 取出。

### 生产生态中的 RL 路径

示例只演示数据流骨架；真实大规模 RL 由下游生态把权重与隐状态搬移推到极致：

- **SGLang P2P 权重传输（Apr 29, 2026）**：SGLang 以 Mooncake TransferEngine 做 RDMA P2P weight transfer，对 **1T 参数 Kimi-K2** 跨数千 GPU 的权重更新从 **53s → 7.2s**（7x），核心是 zero-copy RDMA。见 [SGLang blog](https://lmsys.org/blog/2026-04-29-p2p-update/)（README 同条目）。
- **TorchSpec（Mar 19, 2026）**：[TorchSpec](https://pytorch.org/blog/torchspec-speculative-decoding-training-at-scale/) 用 Mooncake 解耦推理与训练，经高效 hidden states 管理支持大规模 speculative decoding 训练。
- **DSpark（Jul 2, 2026）**：DSpark 在 GB300 NVL72 上以 Speculators + Mooncake 做全在线训练：9 个 vLLM 节点经 Mooncake RDMA Store 把 GLM 5.2 FP8 verifier 供给 6 个 FSDP 训练节点（DP=24），达 125k prefill tokens/s、1.5 steps/s。
- **ROLL（Dec 27, 2025）**：相关 RL 论文 [arXiv 2512.22560](https://arxiv.org/abs/2512.22560)。

这些案例印证了 Mooncake 在 RL 中的两个底层价值：TE 做 **zero-copy 权重 / 隐状态同步**（替代 NCCL broadcast 的串行权重加载），Store 做 **rollout 张量对象的临时共享与生命周期管理**。

## 4. 交叉引用

- 底层搬移：[`../02-transfer-engine/index.md`](../02-transfer-engine/index.md) —— TE 是 RL zero-copy 权重 / 隐状态传输的底座；SGLang P2P weight transfer 直接用 TransferEngine。
- 存储引擎：[`../03-store/index.md`](../03-store/index.md) —— `rl_samples.py` 调用的 `MooncakeDistributedStore.put_tensor` / `get_tensor` 即 Store Python API；DSpark 亦以 Store 做 verifier↔trainer 数据通路。
- 外部生态：[`../11-ecosystem/index.md`](../11-ecosystem/index.md) —— SGLang / TorchSpec / DSpark / ROLL 等下游项目把 RL 场景落地到生产。
- 官方 Store API 参考：[`../../docs/source/python-api-reference/mooncake-store.html`](../../docs/source/python-api-reference/mooncake-store.html)（`MooncakeDistributedStore` 签名与 `setup` 参数）。
- 主仓库 README RL 条目：[`../../README.md`](../../README.md)（Apr 29, 2026 SGLang P2P weight transfer；Mar 19, 2026 TorchSpec；Jul 2, 2026 DSpark）。

## 5. 关键源码入口

- `mooncake-rl/examples/rl_samples.py:1` —— 头注释：dummy RL 训练示例，演示 Mooncake Store 在 rollout↔train 间的数据传输。
- `mooncake-rl/examples/rl_samples.py:7` —— `from mooncake.store import MooncakeDistributedStore`。
- `mooncake-rl/examples/rl_samples.py:10` —— `TrainActor`（单训练 worker）。
- `mooncake-rl/examples/rl_samples.py:76` —— `TrainGroup`（训练引擎组）。
- `mooncake-rl/examples/rl_samples.py:98` —— `MooncakeDistributedStore()` 客户端实例化。
- `mooncake-rl/examples/rl_samples.py:100` —— `setup(...)` RDMA 初始化（segment 512MB / buffer 128MB / NIC）。
- `mooncake-rl/examples/rl_samples.py:125` —— `update_weights`（示例中 mock；真实路径见 SGLang/TorchSpec）。
- `mooncake-rl/examples/rl_samples.py:142` —— `get_tensor(rollout_key)` 从 Store 取 rollout 样本。
- `mooncake-rl/examples/rl_samples.py:167` —— `RolloutEngine`（rollout 推理 worker）。
- `mooncake-rl/examples/rl_samples.py:217` —— `RolloutController`（dataset load/save，含 Store 客户端）。
- `mooncake-rl/examples/rl_samples.py:273` —— `RolloutManager`（聚合 rollout 写 Store）。
- `mooncake-rl/examples/rl_samples.py:306` —— `put_tensor(key, samples)` 写入 Store。
- `mooncake-rl/examples/rl_samples.py:335` —— `train()` 主循环（rollout→store→train→weight update→eval）。
- `mooncake-rl/examples/rl_samples.py:448` —— `__main__` 入口。

## 参考文档

- 官方 Store API：[`../../docs/source/python-api-reference/mooncake-store.html`](../../docs/source/python-api-reference/mooncake-store.html)
- 主仓库 README：[`../../README.md`](../../README.md)（RL 相关更新条目）
- SGLang P2P weight transfer：[blog](https://lmsys.org/blog/2026-04-29-p2p-update/)
- TorchSpec：[blog](https://pytorch.org/blog/torchspec-speculative-decoding-training-at-scale/) · [repo](https://github.com/torchspec-project/TorchSpec)
- ROLL 论文：[arXiv 2512.22560](https://arxiv.org/abs/2512.22560)
