# 附录（Appendix）

## 定位

本页汇总 Mooncake 内部架构 Wiki 的**尾端参考材料**：错误码索引、性能基线、变更时间线、FAQ 指针。错误码与部署细节以**相对链接**指向官方 docs 为主，本页只做高频项的速查汇编，不复制官方内容。性能基线与变更时间线从仓库根 [`../../README.md`](../../README.md) 的 Show Cases 与 Updates 提炼，便于在源码侧快速回溯里程碑。

对应 Wiki 子系统树的最末一节，与 [`../00-overview/`](../00-overview/) 的术语表、[`../10-cross-cutting/`](../10-cross-cutting/) 的工程治理/CI 资产构成"快速入门←→架构内幕←→速查附录"的闭环。

## 目录 / 源码结构

本页为单页附录，无下属文件。引用的外部源点：

```
docs/source/troubleshooting/
├── error-code.md            错误码官方总表（TE + Store 全量）
├── troubleshooting.md       部署/运行时排障官方页
└── pg-ep-troubleshooting.md EP/PG 专属排障
mooncake-transfer-engine/include/error.h   TE 错误码枚举源
mooncake-store/include/types.h             Store 错误码枚举源
.claude/skills/mooncake-troubleshoot/SKILL.md  排障诊断 skill
.claude/skills/mooncake-ci-local/SKILL.md      本地 CI skill
README.md                                   Show Cases / Updates 源
```

## 核心概念

### 1. 错误码速查

完整错误码表以官方 [`../../docs/source/troubleshooting/error-code.md`](../../docs/source/troubleshooting/error-code.md) 为准。TE 错误码定义见 `mooncake-transfer-engine/include/error.h`，Store 错误码定义见 `mooncake-store/include/types.h`。下表仅列运行时高频项与典型成因，详细修复步骤见排障 skill（[`.claude/skills/mooncake-troubleshoot/SKILL.md`](../../../.claude/skills/mooncake-troubleshoot/SKILL.md)）与 [`../../docs/source/troubleshooting/troubleshooting.md`](../../docs/source/troubleshooting/troubleshooting.md)。

| 错误码 / 消息 | 来源 | 含义 | 典型成因 / 修复 |
|--------------|------|------|----------------|
| `ERR_METADATA` (-20) | TE | 与 metadata server 通信失败 | etcd/HTTP metadata server 不可达；检查 `MC_METADATA_SERVER`、`etcd --listen-client-urls http://0.0.0.0:2379`、`unset http_proxy https_proxy` |
| `Error from etcd client`（日志） | Store/TE | 元数据客户端报错 | 同上；etcd 绑定 `127.0.0.1` 而非 `0.0.0.0`、HTTP 代理干扰 |
| `ERR_DNS` (-16) | TE | `local_server_name` 非合法 DNS/IP | 用真实 LAN IP 或有效 hostname，禁用回环 |
| `ERR_DEVICE_NOT_FOUND` (-14) | TE | RDMA 设备名不存在 | `ibv_devices` 取正确设备名 |
| `No matched device found`（日志） | TE | RDMA 设备名与配置不符 | 同上；检查 `MC_MS_FILTERS` |
| `Failed to register memory: Input/output error` | TE | RDMA MR 注册超限 | 检查 `ibv_devinfo -v \| grep max_mr_size`、`ulimit -l unlimited`、`/etc/security/limits.conf` 设 `memlock unlimited` |
| `Failed to create QP: Cannot allocate memory` | TE | QP 数过多 | `export MC_ENABLE_DEST_DEVICE_AFFINITY=1` |
| `Failed to exchange handshake description` | TE | RDMA 握手失败 | 校验 MTU（`MC_MTU`）、GID 非全零（`MC_GID_INDEX=1`），`ib_send_bw` 跨节点预跑 |
| `Failed to modify QP to RTR, check mtu, gid, peer lid, peer qp num` | TE | MTU/GID 不匹配 | 同上 |
| `NO_AVAILABLE_HANDLE` (-200) | Store | 内存池耗尽 | 增大 `global_segment_size`；检查驱逐是否生效 |
| `SEGMENT_NOT_FOUND` (-101) | Store | 无可用 segment | 检查 segment 注册与 `local_hostname` 一致 |
| `OBJECT_NOT_FOUND` (-704) | Store | 对象不存在 | 校验 key |
| `LEASE_EXPIRED` (-707) | Store | 传输前 lease 已过期 | 增大 master 的 `default_kv_lease_ttl` |
| `ETCD_OPERATION_ERROR` (-1000) | Store | etcd 操作失败 | 检查 etcd 健康（`etcdctl endpoint health`） |
| `RPC_FAIL` (-900) | Store | RPC 失败 | 网络或 master 不可达 |
| `bind address already in use`（日志） | Store | 端口冲突 | `--rpc_port=50052` 换端口 |
| EP/PG 相关 | EP/PG | rank 失效、collective 失败 | 见 [`../../docs/source/troubleshooting/pg-ep-troubleshooting.md`](../../docs/source/troubleshooting/pg-ep-troubleshooting.md) |

排障策略（先简后繁、首错优先、`MC_FORCE_TCP=true` 隔离 RDMA、`MC_LOG_LEVEL=0` / `MC_YLT_LOG_LEVEL=debug` 开 verbose）详见 [`../../docs/source/troubleshooting/troubleshooting.md`](../../docs/source/troubleshooting/troubleshooting.md) 与排障 skill。

### 2. 性能基线

下列数字来自 [`../../README.md`](../../README.md) Show Cases，作为源码侧优化与回归对比的公开基线引用。详细测试条件见对应链接。

| 指标 | 数值 | 条件 | 来源 |
|------|------|------|------|
| TE 聚合带宽 | **87 GB/s** | 4×200 Gbps RoCE，40 GB（LLaMA3-70B 128k tokens KVCache） | [README Show Cases](../../README.md#show-cases) |
| TE 聚合带宽 | **190 GB/s** | 8×400 Gbps RoCE，40 GB 同上 | 同上（相对 TCP 分别 2.4× / 4.6×） |
| Kimi 线上增益 | **+75% 请求量** | 真实负载下保持 SLO | [`../../README.md`](../../README.md#overview) |
| Kimi-K2 大规模推理 | **224k prefill / 288k decode tok/s** | 128×H200，PD 分离 + 大规模 EP | [K2 blog](https://lmsys.org/blog/2025-07-20-k2-large-scale-ep/)（2025-07-20） |
| checkpoint-engine 权重更新 | **~20s 更新 1T 参数** | K1.5/K2 训练，数千 GPU | [checkpoint-engine](https://github.com/MoonshotAI/checkpoint-engine/)（2025-09-10） |
| SGLang×Mooncake RL P2P 权重 | **53s → 7.2s（~7×）** | Kimi-K2 1T 参数，零拷贝 RDMA | [SGLang P2P blog](https://lmsys.org/blog/2026-04-29-p2p-update/)（2026-04-29） |
| DSpark GB300 NVL72 在线 RL | **125k prefill tok/s · 1.5 steps/s** | 9 vLLM 推理 + 6 FSDP 训练（DP=24），GLM 5.2 FP8 verifier | [`../../README.md`](../../README.md#updates)（2026-07-02） |

本地存储侧 micro-benchmark 用 `benchmarks/storage_benchmark_v1/`（见 [`../10-cross-cutting/`](../10-cross-cutting/)）；TE 端到端带宽可用 `mooncake-transfer-engine/benchmark/` 内的示例验证。

### 3. 变更时间线（关键里程碑）

完整列表见 [`../../README.md#updates`](../../README.md#updates)。下表为本 Wiki 各子系统源码可对应的里程碑（2024-06 ~ 2026-07）。

| 日期 | 里程碑 | 关联子系统 |
|------|-------|-----------|
| 2024-06-26 | 初始技术报告发布 | 仓库根 |
| 2024-07-09 | 公开 trace JSONL | `FAST25-release/traces` |
| 2024-11-28 | Transfer Engine 开源（含 P2P Store / vLLM 集成 demo） | `02-transfer-engine` / `04-p2p-store` |
| 2024-12-16 | vLLM 官方支持 TE 用于 PD | `11-ecosystem` |
| 2025-02-25 | FAST 2025 **Best Paper** | 仓库根 |
| 2025-02-21 | FAST'25 traces 更新 | `FAST25-release/traces` |
| 2025-03-07 | Mooncake Store 开源 | `03-store` |
| 2025-04-10 | SGLang 官方支持 TE 用于 PD | `11-ecosystem` |
| 2025-04-22 | LMCache 支持 Store 作为 remote connector | `11-ecosystem` |
| 2025-05-05 | SGLang DeepSeek PD 96×H100 指南 | `11-ecosystem` |
| 2025-05-08 | Mooncake × LMCache 联合方案 | `11-ecosystem` |
| 2025-05-09 | NIXL 支持 TE 作为后端插件 | `11-ecosystem` |
| 2025-06-20 | LMDeploy PD backend 上线 | `11-ecosystem` |
| 2025-07-20 | Kimi-K2 128×H200 PD+EP，224k/288k tok/s | `03-store` / `05-ep` / `06-pg` |
| 2025-08-18 | vLLM-Ascend 集成 TE | `11-ecosystem` |
| 2025-08-23 | xLLM 基于 Mooncake 的混合 KV cache | `11-ecosystem` |
| 2025-09-10 | SGLang HiCache（Store 为远端 tier）；checkpoint-engine 开源（1T ~20s）；vLLM-Ascend Store KV pool | `03-store` / `04-p2p-store` / `11-ecosystem` |
| 2025-09-18 | vLLM-Ascend 以 Store 为 KV pool 后端 | `11-ecosystem` |
| 2025-11-07 | RBG + SGLang HiCache + Mooncake 云原生部署 | `11-ecosystem` |
| 2025-12-19 | TensorRT-LLM 集成 TE；vLLM v1 `MooncakeConnector` | `11-ecosystem` / `09-wheel` |
| 2025-12-23 | SGLang EPD 分离（多模态） | `11-ecosystem` |
| 2025-12-27 | ROLL 合作论文 | `11-ecosystem` |
| 2026-01-28 | FlexKV 分布式 KVCache reuse | `11-ecosystem` |
| 2026-02-12 | Mooncake 加入 PyTorch Ecosystem | 仓库根 |
| 2026-02-24 | vLLM-Omni 双 connector | `11-ecosystem` |
| 2026-02-25 | SGLang Encoder Global Cache（多模态 ViT embedding） | `11-ecosystem` |
| 2026-03-05 | LightX2V disaggregated 部署 | `11-ecosystem` |
| 2026-03-19 | TorchSpec 开源（Store 管理 hidden states） | `11-ecosystem` |
| 2026-03-25 | SGLang Elastic EP（部分 rank 容错） | `05-ep` / `06-pg` |
| 2026-04-29 | SGLang P2P 权重传输，Kimi-K2 1T 53s→7.2s | `04-p2p-store` / `11-ecosystem` |
| 2026-05-07 | vLLM 官方 blog 推荐 Mooncake Store | `11-ecosystem` |
| 2026-07-02 | DSpark GB300 NVL72 在线 RL，125k tok/s | `11-ecosystem` |

### 4. FAQ 指针

常见问题的官方/源码侧入口，按主题归类：

1. **`mooncake_master` 启动后 Prometheus/Grafana 看不到数据？** —— 检查 `--metrics_port=9003 --enable_metric_reporting=true`，见 [`../10-cross-cutting/`](../10-cross-cutting/) "监控栈" 与 [`../../docs/source/getting_started/observability.md`](../../docs/source/getting_started/observability.md)。
2. **`Error from etcd client` / `ERR_METADATA`？** —— etcd 绑定 `0.0.0.0`、`unset http_proxy https_proxy`，见 [`../../docs/source/troubleshooting/troubleshooting.md`](../../docs/source/troubleshooting/troubleshooting.md)。
3. **`Failed to register memory` / `Cannot allocate memory`？** —— `ulimit -l unlimited` 与 `memlock` limits，或 `MC_ENABLE_DEST_DEVICE_AFFINITY=1` 限 QP 数。
4. **`No matched device found` / `Failed to exchange handshake`？** —— `ibv_devices` / `ibv_devinfo` 校验设备与端口，`MC_GID_INDEX` / `MC_MTU` 对齐。
5. **`NO_AVAILABLE_HANDLE` (-200) / `LEASE_EXPIRED` (-707)？** —— 增大 `global_segment_size` / `default_kv_lease_ttl`，见 [`../../docs/source/troubleshooting/error-code.md`](../../docs/source/troubleshooting/error-code.md)。
6. **EP/PG rank 失效或 collective 失败？** —— 见 [`../../docs/source/troubleshooting/pg-ep-troubleshooting.md`](../../docs/source/troubleshooting/pg-ep-troubleshooting.md) 与 [`../05-ep/`](../05-ep/)、[`../06-pg/`](../06-pg/)。
7. **提交 PR 前如何本地验证？** —— `bash scripts/run_ci_test.sh`（CI skill 默认入口），见 [`.claude/skills/mooncake-ci-local/SKILL.md`](../../../.claude/skills/mooncake-ci-local/SKILL.md) 与 [`../10-cross-cutting/`](../10-cross-cutting/)。
8. **如何与 SGLang / vLLM 集成？** —— 官方集成指南 [`../../docs/source/getting_started/examples/`](../../docs/source/getting_started/examples/)，集成矩阵见 [`../11-ecosystem/`](../11-ecosystem/)。

## 交叉引用

- 术语表：[`../00-overview/glossary.md`](../00-overview/glossary.md)
- 架构拓扑 / 依赖图：[`../00-overview/architecture.md`](../00-overview/architecture.md) / [`../00-overview/dependency-graph.md`](../00-overview/dependency-graph.md)
- 硬件协议矩阵：[`../00-overview/hardware-protocol-matrix.md`](../00-overview/hardware-protocol-matrix.md)
- 横向支撑（CI / 监控 / 基准 / 治理）：[`../10-cross-cutting/`](../10-cross-cutting/)
- 外部生态集成：[`../11-ecosystem/`](../11-ecosystem/)
- 错误码官方总表：[`../../docs/source/troubleshooting/error-code.md`](../../docs/source/troubleshooting/error-code.md)
- 排障官方页：[`../../docs/source/troubleshooting/troubleshooting.md`](../../docs/source/troubleshooting/troubleshooting.md)
- EP/PG 排障：[`../../docs/source/troubleshooting/pg-ep-troubleshooting.md`](../../docs/source/troubleshooting/pg-ep-troubleshooting.md)
- 可观测性指南：[`../../docs/source/getting_started/observability.md`](../../docs/source/getting_started/observability.md)
- 主仓库 README（Updates / Show Cases 源）：[`../../README.md`](../../README.md)
- 排障 skill：[`.claude/skills/mooncake-troubleshoot/SKILL.md`](../../../.claude/skills/mooncake-troubleshoot/SKILL.md)
- 本地 CI skill：[`.claude/skills/mooncake-ci-local/SKILL.md`](../../../.claude/skills/mooncake-ci-local/SKILL.md)

## 关键源码入口

| 关注点 | 入口 |
|--------|------|
| TE 错误码枚举源 | `mooncake-transfer-engine/include/error.h` |
| Store 错误码枚举源 | `mooncake-store/include/types.h` |
| 错误码官方总表 | `docs/source/troubleshooting/error-code.md` |
| 部署排障官方页 | `docs/source/troubleshooting/troubleshooting.md` |
| EP/PG 排障页 | `docs/source/troubleshooting/pg-ep-troubleshooting.md` |
| 可观测性指南 | `docs/source/getting_started/observability.md` |
| 存储 micro-benchmark | `benchmarks/storage_benchmark_v1/benchmark.py` |
| TE 带宽示例 | `mooncake-transfer-engine/benchmark/` |
| 排障 skill | `.claude/skills/mooncake-troubleshoot/SKILL.md:247`（错误码速查起） |
| 本地 CI skill | `.claude/skills/mooncake-ci-local/SKILL.md:8` |
| Updates / Show Cases 源 | `README.md`（`#updates` / `#show-cases` 锚点） |
