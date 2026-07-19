# 子系统依赖与交叉引用

## 依赖层次（自底向上）

```
              ┌──────────────────────────────────────────────────┐
   分发层     │ mooncake-wheel  (pip 包, 打包 integration+store+ep+pg) │
              └───────┬───────────────────────────────┬──────────┘
                      │                                │
              ┌───────▼──────────┐         ┌───────────▼──────────┐
   绑定层     │ mooncake-        │         │ mooncake-rl           │
              │ integration      │         │ (示例, 直接用 TE/Store)│
              └───┬──────────┬───┘         └──────────────────────┘
                  │          │
        ┌─────────▼──┐   ┌──▼──────────────────────┐
   执行层│ mooncake-ep│   │ mooncake-store          │  存储层
        │            │   │                         │
        └─────┬──────┘   └────────┬────────────────┘
              │                   │
        ┌─────▼──────┐   ┌────────▼────────┐
        │ mooncake-pg│   │ mooncake-p2p    │
        │ (依赖 EP)  │   │ -store          │
        └─────┬──────┘   └────────┬────────┘
              │                   │
              └────────┬──────────┘
                       │
              ┌────────▼─────────────────────┐
   传输层     │ mooncake-transfer-engine (+tent)│
              └────────┬─────────────────────┘
                       │
              ┌────────▼─────────────┐
   公共层     │ mooncake-common       │
              │ (cmake/etcd/k8s-lease)│
              └──────────────────────┘
```

关键依赖事实：

- **common ← 一切**：所有子系统 `include_directories(mooncake-common/include)`（`CMakeLists.txt:73`）。
- **TE ← Store / P2P / EP / PG / RL / integration**：TE 是通用数据搬移底座；Store 的 `transfer_task/task_manager`、EP/PG 的底层搬移均走 TE API。
- **Store ← common + TE**：`WITH_STORE_RUST` 要求 `WITH_STORE=ON`（`CMakeLists.txt:88`）。
- **EP ← TE(engine.so) + PyTorch**：`mooncake_ep_ext` 自定义目标 `DEPENDS engine`（`CMakeLists.txt:154`）。
- **PG ← TE + EP**：`mooncake_pg_ext` `DEPENDS engine mooncake_ep_ext`（`CMakeLists.txt:173`）；`06-pg` 依赖 `05-ep`。
- **integration ← TE + Store**：`mooncake-integration` 作为独立 `add_subdirectory`（`CMakeLists.txt:179`），同时绑定 TE 与 Store 扩展。
- **wheel ← integration + Store + EP + PG**：最终 pip 轮子聚合 Python 包，是整栈的打包出口。

## 横向交叉（跨子系统共享关注点）

### 1. 异构硬件后端横向复用

`TE/transport/*`（传输）与 `Store/device/accelerator_*`（加速器内存注册）对接**同一批异构硬件**，是跨子系统的共享维度：

| 硬件家族 | TE transport | TE allocator | Store device |
|---------|-------------|-------------|-------------|
| NVIDIA CUDA | rdma/tcp/nvlink | cuda_alike · ub_allocator | cuda_like_accelerator_device |
| AMD HIP | hip_transport | hip_device_guard · gpu_vendor | hip_accelerator_device |
| 华为 Ascend NPU | ascend_transport + ascend_direct | ascend_allocator | ascend_accelerator_device |
| 寒武纪 MLU | maca_transport | (gpu_vendor) | — |
| 摩尔线程 MUSA | (cuda_alike) | (cuda_alike) | cuda_like_accelerator_device |
| Sunrise | sunrise_link_transport | sunrise_allocator | sunrise_accelerator_device |
| 昆鹏 UB | kunpeng_transport | (ub_allocator) | — |

详见硬件协议矩阵：[`hardware-protocol-matrix.md`](hardware-protocol-matrix.md)。

### 2. 元数据 / HA 横向

`common/etcd` + `common/k8s-lease` 在多个子系统复用：

```
mooncake-common/etcd  ─┐
mooncake-common/k8s-lease ─┤
                          ├── mooncake-store: etcd_helper / k8s_lease_helper
                          │                    http_metadata_server / metadata_store
                          ├── mooncake-store: ha/ (leadership/oplog/snapshot/standby)
                          └── mooncake-store: master HA 后端 (etcd vs K8s Lease 互斥, CMakeLists.txt:50-58)
```

注意构建约束：`STORE_USE_K8S_LEASE` 与 `STORE_USE_ETCD` **不能同时开启**（`CMakeLists.txt:51`），因两者均构建 Go c-shared HA 后端。

### 3. TE 作为通用底座

TE 的 `TransferEngine` API 被多处调用：

- **Store**：`transfer_task.cpp` / `task_manager.cpp` —— 主从节点间 Segment 数据搬移。
- **P2P Store**：peer 间直接传输。
- **EP/PG**：底层 collective / dispatch/combine 搬移。
- **RL 示例**：权重与隐状态传输。
- **integration**：`transfer_engine_py.cpp` 暴露给 Python。

### 4. Engram ↔ Conductor

Store 的 `src/engram/engram_store.cpp` 是 **Engram embedding 表存储后端**（per-head 表的 populate/lookup），**不是** `design/conductor/conductor-architecture-design.md` 描述的 Conductor 索引的实现。Conductor 是独立 Go 项目的全局 KV-cache 前缀索引服务，二者关系是"KV 事件发布者(Store master) ↔ 消费者(Conductor)"而非"实现 ↔ 设计"。详见 [`../03-store/engram-conductor.md`](../03-store/engram-conductor.md)。

### 5. TENT 是 TE 的下一代演进

`mooncake-transfer-engine/tent/` 作为独立子树，引入声明式 slice spraying、failover、QoS、transport selector，对原 TE 的 `multi_transport` 做能力增强而非替换。两者并存，详见 [`../02-transfer-engine/tent.md`](../02-transfer-engine/tent.md)。

### 6. Python 全链路三层递进

```
C++ 内核 (store/te/ep/pg)
   │  pybind11
   ▼
mooncake-integration/  (store_py.cpp / transfer_engine_py.cpp 等扩展模块)
   │  import
   ▼
mooncake-wheel/mooncake/  (纯 Python 包: cli / connector / service 包装)
   │  pip install mooncake-transfer-engine
   ▼
外部生态 (SGLang / vLLM / ...)
```

`extern/pybind11` 作为 git 子模块提供绑定机制（`CMakeLists.txt:25`）。

## 构建开关矩阵

| CMake option | 控制子系统 | 依赖约束 |
|--------------|-----------|---------|
| `WITH_TE` | transfer-engine | 默认 ON |
| `WITH_STORE` | store | 默认 ON |
| `WITH_STORE_RUST` | store/rust | 需 `WITH_STORE=ON` |
| `WITH_STORE_GO` | store/go | 需 `WITH_STORE=ON` |
| `WITH_P2P_STORE` | p2p-store | 默认 OFF |
| `WITH_EP` | ep + pg | EP 需 CUDA + PyTorch |
| `WITH_RUST_EXAMPLE` | te/rust | 默认 OFF |
| `STORE_USE_ETCD` | store 元数据 | 与 K8S_LEASE 互斥 |
| `STORE_USE_REDIS` | store 元数据 | — |
| `STORE_USE_K8S_LEASE` | store HA | 与 etcd 互斥 |
| `USE_NOF` | store SSD pool | — |

## 参考文档

- 顶层构建入口：`CMakeLists.txt`
- 官方架构：`../../docs/source/design/architecture.md`
- 硬件矩阵：[`hardware-protocol-matrix.md`](hardware-protocol-matrix.md)
- 术语：[`glossary.md`](glossary.md)
