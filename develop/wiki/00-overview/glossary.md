# 术语表

本 Wiki 使用的核心术语与缩写，附源码/文档定位。

| 术语 | 全称 / 含义 | 源码/文档定位 |
|------|------------|--------------|
| Mooncake | KVCache 为中心的 disaggregated LLM 推理/训练基础设施 | 仓库根 |
| KVCache | Transformer 推理的 Key/Value 缓存，Mooncake 的一等公民对象 | — |
| PD Disaggregation | Prefill-Decode 分离：prefill 与 decode 集群解耦 | docs 多篇 |
| SLO | Service Level Objective，延迟/吞吐服务目标 | `docs/source/index.md` |
| TE | Transfer Engine，mooncake-transfer-engine 子系统 | `mooncake-transfer-engine/` |
| TENT | 下一代 TE：声明式 slice spraying 引擎 | `mooncake-transfer-engine/tent/` |
| Store | Mooncake Store，分布式 KVCache/权重存储引擎 | `mooncake-store/` |
| P2P Store | mooncake-p2p-store，去中心化 peer 存储；生产版为 checkpoint-engine | `mooncake-p2p-store/` |
| EP | Expert Parallelism，专家并行（MoE） | `mooncake-ep/` |
| PG | Process Group，torch.distributed 后端 | `mooncake-pg/` |
| Segment | Store 中存储对象的物理分片单元，数据搬移的粒度 | `mooncake-store/src/segment.cpp` |
| Engram | Store 内的 embedding 表存储后端（per-head 表 populate/lookup） | `mooncake-store/src/engram/` |
| Conductor | 独立 Go 项目的全局 KV-cache 前缀索引服务，订阅 Store KV 事件 | `docs/source/design/conductor/`（设计文档；实现不在本仓库） |
| Master | Store 控制面节点，管理对象到 buffer 映射 | `mooncake-store/src/master.cpp` |
| Managed Pool / Buffer Node | 提供 DRAM 存储的受管节点 | `docs/source/design/architecture.md` |
| ReplicateConfig | 对象级复制策略（副本数/偏好段/soft pin/hard pin） | Store API |
| HA | High Availability，Store master 主备高可用 | `mooncake-store/src/ha/` |
| Slice Spraying | TENT 的声明式切片分发策略 | `docs/source/design/tent/slice-spraying.md` |
| active_ranks | EP 中感知失效 rank 的 MoE 容错机制 | `mooncake-ep/` |
| Tensor-centric | Mooncake 全栈以 Tensor 为基础数据载体的设计哲学 | `README.md` |
| HiCache | SGLang 分层 KV 缓存，Store 作为外部存储后端 | `docs/source/design/hicache-design.md` |
| KV Event | 对象变更事件发布（kv_event_publisher） | `mooncake-store/src/kv_event/` |
| NoF | NVMe-oF，远端 SSD pool（`USE_NOF`） | `CMakeLists.txt:22` |
| RL | Reinforcement Learning，RL 训练场景的权重/隐状态传输 | `mooncake-rl/` |
| RadixAttention | SGLang 前缀缓存树，Store 扩展其层级 | SGLang 侧 |
| Disaggregated Serving | 推理算力解耦（prefill/decode/encode 等分离） | docs 多篇 |

## 缩写 cheatsheet

- **TE** Transfer Engine
- **TENT** TE Next-gen (slice spraying)
- **EP** Expert Parallelism
- **PG** Process Group
- **HA** High Availability
- **PD** Prefill-Decode
- **MoE** Mixture of Experts
- **NPU** Neural Processing Unit (Ascend 等)
- **NIC** Network Interface Card
- **RDMA** Remote Direct Memory Access
- **NoF** NVMe over Fabrics
- **CXL** Compute Express Link
- **EFA** Elastic Fabric Adapter (AWS)
