# 外部生态集成（External Ecosystem）

## 定位

Mooncake 的对外价值不在自身单跑，而在于**作为 Tensor 搬移与分布式存储底座嵌入主流推理/训练框架**。本页以 Mooncake 主仓库 `README.md` 的 Updates 时间线与 Show Cases 为源头，归纳 Mooncake 与外部框架/系统的集成关系，给出**集成对象 × 集成方式 × 场景 × 关键时间 × 参考链接**的对照表，并用三张分层图说明接缝位置。

集成方式对照本 Wiki 的子系统分层（见 [`../00-overview/dependency-graph.md`](../00-overview/dependency-graph.md)）：

- **TE**：直接调用 `TransferEngine` API 或作为后端插件
- **Store**：以 `MooncakeStoreConnector` / 远端 connector 形式接入 Mooncake Store
- **Connector**：在框架自身的 KV Connector 抽象中分发到 Mooncake（在 wheel 层以 Python 模块形式交付，见 [`../09-wheel/`](../09-wheel/)）
- **EP+PG**：Mooncake EP / PG 作为 MoE collective / `torch.distributed` 后端
- **P2P-TE**：基于 TE 的 P2P 权重/隐状态传输（生产版为 checkpoint-engine）
- **HiCache**：Store 作为分层 KV 缓存的远端 tier

## 目录 / 源码结构

本仓库内与生态集成直接相关的源点（外部集成代码本身位于各伙伴仓库）：

```
mooncake-wheel/mooncake/
├── mooncake_connector_v1.py     vLLM v1 MooncakeConnector（随 wheel 分发）
├── vllm_v1_proxy_server.py      vLLM 集成测试用 proxy server
└── mooncake_store_service.py    Store 服务封装（被 connector 复用）
mooncake-rl/                     RL 场景示例（权重/隐状态传输）
mooncake-transfer-engine/rust/ · mooncake-store/rust/   Rust 客户端（NIXL 等可经 FFI 复用）
mooncake-transfer-engine/src/transfer_engine_c.cpp      C ABI（Go/Rust/Python FFI 底座）
docs/source/getting_started/examples/
├── sglang-integration-v1.md     SGLang 集成指南
├── sglang-integration/          SGLang 子文档
├── vllm-integration/            vLLM 子文档
├── lmcache-integration.md       LMCache 集成指南
└── lmdeploy-integration-v0.9.md LMDeploy 集成指南
```

外部生态代码的接缝分三层（自底向上）：

```
        外部框架（SGLang / vLLM / TRT-LLM / LMCache / NIXL / ...）
                          │
              ┌───────────┴────────────┐
   Python    │ MooncakeConnector /    │  Connector 层（vLLM v1 / SGLang）
   连接器    │ HiCache 远端 tier /    │  HiCache 层（SGLang HiCache）
              │ KV pool 后端           │  KV pool 层（vLLM-Ascend / LMCache）
              └───────────┬────────────┘
                          │  import mooncake.*
              ┌───────────▼────────────┐
   wheel 层  │ mooncake-wheel/         │  见 ../09-wheel/
              │ mooncake_connector_v1  │
              └───────────┬────────────┘
                          │  pybind11 / C ABI
              ┌───────────▼────────────┐
   内核      │ TE + Store + EP + PG    │  见 ../02 ~ ../06
              └────────────────────────┘
```

## 核心概念

### 集成对象总表

按场景归类，覆盖 `README.md` 截至 2026-07 的 Updates 与 Show Cases 全部条目。

| 集成对象 | 集成方式 | 场景 | 关键日期 | 参考链接 |
|---------|---------|------|---------|---------|
| **vLLM** | TE | PD-disagg | 2024-12-16 | [README Updates](../../README.md#updates) |
| **vLLM** | Connector（v1 `MooncakeConnector`） | PD-disagg KV transfer | 2025-12-19 | [vLLM docs](https://docs.vllm.ai/en/latest/features/mooncake_connector_usage/) |
| **vLLM** | Store（`MooncakeStoreConnector`） | 跨实例 KV pool 共享 | 2026-05-07 | [vLLM blog](https://vllm.ai/blog/mooncake-store) |
| **vLLM-Omni** | Connector × 2 | 多阶段 omni 流水线 | 2026-02-24 | [vLLM-Omni design](https://docs.vllm.ai/projects/vllm-omni/en/latest/design/feature/disaggregated_inference/) |
| **vLLM-Ascend** | TE | Ascend NPU PD-prefill | 2025-08-18 | [vLLM-Ascend guide](https://docs.vllm.ai/projects/ascend/en/latest/developer_guide/feature_guide/disaggregated_prefill.html) |
| **vLLM-Ascend** | Store | NPU 分布式 KV pool 后端 | 2025-09-18 | [vLLM-Ascend KV pool](https://docs.vllm.ai/projects/ascend/zh-cn/main/user_guide/feature_guide/kv_pool.html) |
| **SGLang** | TE | PD-disagg KV transfer | 2025-04-10 | [SGLang guide](../../docs/source/getting_started/examples/sglang-integration-v1.md) |
| **SGLang** | Store（HiCache） | HiCache 远端 tier | 2025-09-10 | [SGLang HiCache blog](https://lmsys.org/blog/2025-09-10-sglang-hicache/) |
| **SGLang** | EP+PG | 容错 EP（Elastic EP） | 2026-03-25 | [Elastic EP blog](https://www.lmsys.org/blog/2026-03-25-eep-partial-failure-tolerance/) |
| **SGLang** | TE | 多模态 EPD（Encode-Prefill-Decode） | 2025-12-23 | [EPD blog](https://lmsys.org/blog/2026-01-12-epd/) |
| **SGLang** | TE | 多模态 Encoder Global Cache（ViT embedding） | 2026-02-25 | [SGLang PR #16137](https://github.com/sgl-project/sglang/pull/16137) |
| **SGLang-Omni** | TE（relay） | omni 多阶段 tensor/blob 中继 | 2026 Q1 | [sglang-omni](https://github.com/sgl-project/sglang-omni) |
| **SGLang** | P2P-TE | RL P2P 权重传输（Kimi-K2 1T：53s→7.2s） | 2026-04-29 | [SGLang P2P blog](https://lmsys.org/blog/2026-04-29-p2p-update/) |
| **TensorRT-LLM** | TE | PD-disagg KV transfer | 2025-12-19 | [TRT-LLM mooncake_utils](https://github.com/NVIDIA/TensorRT-LLM/tree/main/cpp/tensorrt_llm/executor/cache_transmission/mooncake_utils) |
| **LMCache** | Store | 远端 connector | 2025-04-22 | [LMCache blog](https://blog.lmcache.ai/2025-04-22-tencent/) |
| **LMCache × Mooncake** | Store | 联合 KVCache 服务系统 | 2025-05-08 | [LMCache integration guide](../../docs/source/getting_started/examples/lmcache-integration.md) |
| **LMDeploy** | TE | PD disagg backend | 2025-06-20 | [LMDeploy guide](../../docs/source/getting_started/examples/lmdeploy-integration-v0.9.md) |
| **NIXL** | TE | 后端插件 | 2025-05-09 | [NIXL mooncake plugin](https://github.com/ai-dynamo/nixl/blob/main/src/plugins/mooncake/README.md) |
| **checkpoint-engine** | P2P-TE（生产版 P2P Store） | K1.5/K2 训练权重更新（1T 参数 ~20s） | 2025-09-10 | [checkpoint-engine](https://github.com/MoonshotAI/checkpoint-engine/) · 见 [`../04-p2p-store/`](../04-p2p-store/) |
| **TorchSpec** | Store | Speculative decoding 训练 hidden states 管理 | 2026-03-19 | [TorchSpec blog](https://pytorch.org/blog/torchspec-speculative-decoding-training-at-scale/) · [repo](https://github.com/torchspec-project/TorchSpec) |
| **FlexKV** | TE | 分布式 KVCache reuse | 2026-01-28 | [FlexKV dist reuse](https://github.com/taco-project/FlexKV/blob/main/docs/dist_reuse/README_en.md) |
| **xLLM** | Store | 混合 KV cache 管理（智能 offload/prefetch） | 2025-08-23 | [xLLM](https://github.com/jd-opensource/xllm) |
| **ROLL** | —（合作论文） | 大规模 RL 训练架构 | 2025-12-27 | [ROLL paper](https://arxiv.org/abs/2512.22560) · [repo](https://github.com/alibaba/ROLL) |
| **RBG** | Store × SGLang HiCache | 云原生 role-based 部署 | 2025-11-07 | [RBG kep #74](https://github.com/sgl-project/rbg/blob/main/keps/74-mooncake-integration/README.md) |
| **LightX2V** | TE | 视频模型 disaggregated 部署（encoder/transformer 解耦） | 2026-03-05 | [LightX2V PR #893](https://github.com/ModelTC/LightX2V/pull/893) · [blog](https://light-ai.top/LightX2V-BLOG/posts/Disaggregation/) |
| **TransferQueue** | Store | 异步状态搬移，解耦推理/训练/RL | 见 README Show Cases | [TransferQueue](https://github.com/Ascend/TransferQueue) |
| **DSpark** | Store（RDMA）+ TE | GB300 NVL72 全在线 RL 训练（125k prefill tok/s） | 2026-07-02 | [DSpark status](https://x.com/mgoin_/status/2072785822231728363) |
| **Kimi-K2 部署** | Store + EP+PG + TE | 128×H200 PD + 大规模 EP（224k/288k tok/s） | 2025-07-20 | [K2 blog](https://lmsys.org/blog/2025-07-20-k2-large-scale-ep/) |

### 场景维度归纳

按 LLM 服务生命周期再聚合：

- **PD 分离推理**（PD-disagg）：vLLM / SGLang / TensorRT-LLM / LMDeploy / vLLM-Ascend — 通过 TE 或 Connector 把 prefill 与 decode 解耦，KV cache 跨节点搬移。
- **分层/共享 KV cache**（HiCache / KV pool）：SGLang HiCache、vLLM Store connector、vLLM-Ascend KV pool、LMCache、FlexKV、xLLM — Store 作为远端 tier 或分布式池，扩展 RadixAttention/前缀缓存的有效容量。
- **大规模 MoE 容错**（EP / EP+PG）：SGLang Elastic EP — Mooncake EP 的 `active_ranks` + PG 的 rank 恢复支撑生产级 EP（见 [`../05-ep/`](../05-ep/)、[`../06-pg/`](../06-pg/)）。
- **多模态 / omni 流水线**（multimodal EPD / omni）：SGLang EPD、SGLang-Omni、vLLM-Omni、LightX2V — TE 负责大体积 embedding / 跨阶段 tensor 的零拷贝中继。
- **RL 训练权重/状态传输**（RL weight / training）：SGLang P2P weight、checkpoint-engine、TorchSpec、TransferQueue、ROLL、DSpark — Store 或 P2P-TE 把权重/hidden states 在推理↔训练集群间搬移，Kimi-K2 1T 权重更新从 53s 降至 7.2s。
- **Checkpoint engine 训练侧**：checkpoint-engine 是 P2P Store 的生产开源版（见 [`../04-p2p-store/`](../04-p2p-store/)）。

### 支持硬件与云厂商矩阵

[`../../README.md`](../../README.md) "Supported Hardware" 列出的硬件/云厂商同时也是生态伙伴：

- **加速器厂商**：NVIDIA、Huawei（Ascend）、AMD（HIP/ROCm）、Cambricon（MLU/MACA）、Moore Threads（MUSA）、Sunrise、Hygon（昆鹏 UB）、MetaX、T-Head
- **云/网络**：AWS（EFA）、Aliyun、Tencent（FlexKV 联合）、NVIDIA（TensorRT-LLM、TorchSpec 共建）

各厂商对应的 TE transport / allocator / Store device 映射见 [`../00-overview/hardware-protocol-matrix.md`](../00-overview/hardware-protocol-matrix.md)。

### 官方集成指南

[`../../docs/source/getting_started/examples/`](../../docs/source/getting_started/examples/) 目录维护 Mooncake 官方维护的集成指南：

- SGLang：`sglang-integration-v1.md` + `sglang-integration/` 子目录
- vLLM：`vllm-integration/` 子目录
- LMCache：`lmcache-integration.md`
- LMDeploy：`lmdeploy-integration-v0.9.md`

## 交叉引用

- connector 随 wheel 分发：[`../09-wheel/`](../09-wheel/)（`mooncake_connector_v1.py` 等）
- Python C++ 扩展桥：[`../08-integration/`](../08-integration/)
- TE API（多 backend 集成底座）：[`../02-transfer-engine/`](../02-transfer-engine/)
- Store 作为远端 tier / KV pool：[`../03-store/`](../03-store/)
- EP / PG（MoE 容错集成）：[`../05-ep/`](../05-ep/) / [`../06-pg/`](../06-pg/)
- P2P Store / checkpoint-engine：[`../04-p2p-store/`](../04-p2p-store/)
- RL 场景示例：[`../07-rl/`](../07-rl/)
- 子系统依赖层次：[`../00-overview/dependency-graph.md`](../00-overview/dependency-graph.md)
- 架构定位（Tensor-centric ecosystem）：[`../00-overview/architecture.md`](../00-overview/architecture.md)
- 官方集成指南目录：[`../../docs/source/getting_started/examples/`](../../docs/source/getting_started/examples/)
- 主仓库 README Updates：[`../../README.md`](../../README.md#updates)
- 性能基线与变更时间线汇总：[`../12-appendix/`](../12-appendix/)

## 关键源码入口

| 关注点 | 入口 |
|--------|------|
| vLLM v1 MooncakeConnector | `mooncake-wheel/mooncake/mooncake_connector_v1.py:1`（用法注释在 `:4-5`） |
| vLLM 集成 proxy server | `mooncake-wheel/mooncake/vllm_v1_proxy_server.py:3` |
| Store 服务 Python 封装 | `mooncake-wheel/mooncake/mooncake_store_service.py` |
| RL 权重/隐状态传输示例 | `mooncake-rl/examples/rl_samples.py`（见 [`../07-rl/`](../07-rl/)） |
| C ABI（Go/Rust/Python FFI 底座） | `mooncake-transfer-engine/src/transfer_engine_c.cpp` |
| Rust 客户端（NIXL 等可复用） | `mooncake-transfer-engine/rust/` · `mooncake-store/rust/` |
| EP/PG 集成（SGLang Elastic EP） | `mooncake-ep/` 与 `mooncake-pg/`（见 [`../05-ep/`](../05-ep/)、[`../06-pg/`](../06-pg/)） |
| 官方集成指南目录 | `docs/source/getting_started/examples/`（SGLang/vLLM/LMCache/LMDeploy） |
