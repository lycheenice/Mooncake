# Transport 后端族（Transport Backends）

## 定位

`mooncake-transfer-engine/src/transport/` 是 TE 的传输后端插件目录。每个子目录是一个独立后端，继承 `Transport` 基类（`include/transport/transport.h:44`），实现 `install / submitTransferTask / getTransferStatus` 等原语。`MultiTransport::installTransport`（`../src/multi_transport.cpp:314`）按 `proto` 名 + `#ifdef USE_*` 编译开关实例化对应后端。

后端数量众多，按互连协议与硬件家族分成四组：通用网络、节点内、远端存储、异构加速器。另有一个 `device/` 目录提供 EP 专用的 P2P / IBGDA device transport，以及 `rpc_communicator/` 作为信令通道。

## 目录 / 源码结构

```
mooncake-transfer-engine/src/transport/
├── transport.cpp                 后端注册基类胶水
├── rdma_transport/               RDMA/RoCE/IB（rdma_context/endpoint/worker_pool/endpoint_store）
├── tcp_transport/                TCP 数据面
├── efa_transport/                AWS EFA（libfabric，efa_context/endpoint）
├── barex_transport/              BaReX/ACCL（soe/solar NIC）
├── cxi_transport/                HPE Slingshot CXI（libfabric）
├── rpc_communicator/             内部 RPC 信令通道（非数据面 transport）
├── nvlink_transport/             跨节点 NVLink（MNNVL）
├── intranode_nvlink_transport/   单节点内 NVLink/PCIe P2P
├── nvmeof_transport/             GPUDirect Storage（cuFile）
├── cxl_transport/                CXL 共享内存
├── ascend_transport/             华为 Ascend NPU（四子类）
│   ├── ascend_direct_transport/  Ascend RoCE 直连
│   ├── hccl_transport/           HCCS via HCCL
│   ├── heterogeneous_rdma_transport/  异构 RDMA
│   └── ubshmem_transport/        UB 共享内存
├── hip_transport/                AMD ROCm（event_pool/stream_pool）
├── maca_transport/               寒武纪 MLU
├── sunrise_link/                 Sunrise/TPU（tang runtime）
├── kunpeng_transport/            昆鹏 UB/Urma（ub_allocator/ub_context/urma）
└── device/                       EP 专用 device transport（P2P + IBGDA mlx5gda）
```

## 核心概念

### 后端共性

所有后端继承 `Transport`，对外接口一致：

- `install(local_server_name, metadata, topology)`：初始化资源（context/QP pool/shm handle 等），向 `TransferMetadata` 注册本段 buffer。
- `submitTransferTask(task_list)`：把 slices 投递到硬件（QP/cuFile/IPC handle），非阻塞。
- `getTransferStatus(batch_id, task_id, status)`：轮询完成，更新 slice 状态，触发重试/失败。

差异在**握手与编址**：RDMA 系用 QP+lkey/rkey，libfabric 系（EFA/CXI）用 64-bit fi_mr_key（`transfer_metadata.h:66-70`），NVLink/HIP 用 shm_name + IPC handle，CXL 用 offset，NVMe-oF 用 file_path+local_path_map。

### 选择与配置

`MultiTransport::selectTransport`（`multi_transport.cpp:452`）根据目标段 `SegmentDesc::protocol` 选后端。全局参数集中在 `GlobalConfig`（`config.h:36`），如 `slice_size=65536`、`num_qp_per_ep`、`gid_index`、`ib_service_level`、`mlx5_qp_udp_sports`（ECMP path diversification）等。各后端另有专属环境变量（`MC_RDMA_*` / `MC_EFA_*` / `ACCL_USE_NICS` 等）。

## 后端矩阵

### 通用网络

| 后端目录 | proto | 适用协议/硬件 | 关键文件 | 特性 |
|---------|-------|--------------|---------|------|
| `rdma_transport` | `rdma` | InfiniBand / RoCEv2 / iWARP（mlx5 等） | `rdma_transport.cpp` / `rdma_context.cpp` / `rdma_endpoint.cpp` / `worker_pool.cpp` / `endpoint_store.cpp` | 多 NIC 带宽聚合、slice 并行、endpoint SIEVE 缓存、rail 失败冷却、ECMP/SL/LAG 调优；TE 默认主力后端 |
| `tcp_transport` | `tcp` | 任意 IP 网络 | `tcp_transport.cpp` | 兜底后端；v2 帧式 WRITE+ACK（`tcp_proto_version`，`transfer_metadata.h:127`）；TCP-only 时同主机走 memcpy |
| `efa_transport` | `efa` | AWS Elastic Fabric Adapter（libfabric） | `efa_transport.cpp` / `efa_context.cpp` / `efa_endpoint.cpp` | 64-bit MR key；EFA endpoint 地址 hex 编码（`HandShakeDesc.efa_addr`）；详见 [`../../docs/source/design/transfer-engine/efa_transport.md`](../../docs/source/design/transfer-engine/efa_transport.md) |
| `barex_transport` | `barex` | ACCL/BaReX（Superluminal soe/solar NIC） | `barex_transport.cpp` / `barex_context.cpp` | 按 topo 过滤 eic NIC，设 `ACCL_USE_NICS`；`USE_BAREX` 编译开关 |
| `cxi_transport` | `cxi` | HPE Slingshot CXI（libfabric） | `cxi_transport.cpp` / `cxi_context.cpp` / `cxi_endpoint.cpp` | 64-bit MR key；与 CUDA/MUSA/MACA 互斥（`transfer_engine.h:31` 的 device transport 守卫）；`cxi_addr` 握手字段 |
| `rpc_communicator` | — | TCP RPC 信令 | `rpc_communicator.cpp` / `rpc_interface.cpp` | 元数据/握手/notify 的 RPC 传输，非数据搬移后端；`RpcCommunicatorConfig`（`config.h:130`） |

### 节点内

| 后端目录 | proto | 适用协议/硬件 | 关键文件 | 特性 |
|---------|-------|--------------|---------|------|
| `nvlink_transport` | `nvlink` | NVIDIA NVLink（MNNVL，跨节点 fabric） | `nvlink_transport.cpp` | `USE_MNNVL`；VMM fabric handle 导出/导入；`gpu_vendor/mnnvl.h` 提供 `allocateFabricMemory` 宏桥接到 hip/ubshmem |
| `intranode_nvlink_transport` | `nvlink_intra` | 单节点内 NVLink / PCIe P2P（CUDA/HIP shim） | `intranode_nvlink_transport.cpp` | `USE_INTRA_NVLINK`；shm + CUDA IPC；节点内 GPU-GPU/GPU-CPU fast-path；`gpu_vendor/intra_nvlink.h` |

### 远端存储

| 后端目录 | proto | 适用协议/硬件 | 关键文件 | 特性 |
|---------|-------|--------------|---------|------|
| `nvmeof_transport` | `nvmeof` | GPUDirect Storage（NVIDIA cuFile）+ NVMe-oF | `nvmeof_transport.cpp` / `cufile_context.cpp` / `cufile_desc_pool.cpp` | 直读 NVMe 到 DRAM/VRAM 零拷贝；`NVMeoFBufferDesc`（file_path+local_path_map，`transfer_metadata.h:81`）；NVMeof Segment 类型 |
| `cxl_transport` | `cxl` | CXL 共享内存 | `cxl_transport.cpp` | `USE_CXL`；用 `BufferDesc.offset`（`transfer_metadata.h:74`）+ `SegmentDesc.cxl_base_addr`（`transfer_metadata.h:117`）编址；多协议段常与 rdma/tcp 并列 |

### 异构加速器

| 后端目录 | proto | 适用协议/硬件 | 关键文件 | 特性 |
|---------|-------|--------------|---------|------|
| `ascend_transport` | `ascend` | 华为 Ascend NPU（HCCS / RoCE / UB shm） | `ascend_direct_transport/` / `hccl_transport/` / `heterogeneous_rdma_transport/` / `ubshmem_transport/` | 由 `USE_ASCEND_DIRECT` / `USE_ASCEND` / `USE_ASCEND_HETEROGENEOUS` / `USE_UBSHMEM` 四选一实例化同一 `ascend` proto；`RankInfoDesc`（`transfer_metadata.h:89`）携带 deviceLogicId/PhyId/Ip/Port；`ascend_store_te_init` 区分 Store vs P2P 资源配置。详见 [`../../docs/source/design/transfer-engine/ascend_transport.md`](../../docs/source/design/transfer-engine/ascend_transport.md) / [`ascend_direct_transport.md`](../../docs/source/design/transfer-engine/ascend_direct_transport.md) / [`heterogeneous_ascend.md`](../../docs/source/design/transfer-engine/heterogeneous_ascend.md) |
| `hip_transport` | `hip` | AMD ROCm GPU | `hip_transport.cpp` / `event_pool.cpp` / `stream_pool.cpp` | `USE_HIP`；HIP IPC / shareable handle；仅同主机有效（`isHipReachableTarget`，`multi_transport_locality.h:74`）；`HipDeviceGuard`（`hip_device_guard.h`）守 device；`gpu_vendor/hip.h` |
| `maca_transport` | `maca` | 寒武纪 MLU（MC runtime） | `maca_transport.cpp` | `USE_MACA`；CUDA-like API 面，`gpu_vendor/maca.h` 做 CU→MC 宏映射；参与 device transport 守卫 |
| `sunrise_link` | `sunrise_link` | Sunrise/TPU（tang runtime） | `sunrise_link_transport.cpp` | `USE_SUNRISE`；`sunrise_allocator.h` + `sunrise_use_device_mem`；详见 [`../../docs/source/design/transfer-engine/sunrise_link_transport.md`](../../docs/source/design/transfer-engine/sunrise_link_transport.md) |
| `kunpeng_transport` | `ub` | 昆鹏 UB/Urma（海光 jfc/jfce/jetty） | `ub_transport.cpp` / `ub_context.cpp` / `ub_allocator.cpp` / `urma/` | `USE_UB`；`BufferDesc.tseg/l_seg_index`（`transfer_metadata.h:75-76`）；`HandShakeDesc.jetty_num`；`ub_allocator`；详见 [`../../docs/source/design/transfer-engine/kunpeng_ub_transport.md`](../../docs/source/design/transfer-engine/kunpeng_ub_transport.md) |
| `device` | — | EP 专用 IBGDA + P2P device transport | `ibgda_device_transport.cpp` / `mlx5gda.cpp` / `p2p_device_transport.cpp` | 非 `MultiTransport` 管理的后端；由 `TransferEngine::getOrCreateP2pTransport` / `getOrCreateRdmaTransport`（`transfer_engine.h:175`）懒创建，供 EP dispatch/combine 用 |

## 交叉引用

- 引擎核心与选择机制：[`index.md`](index.md)
- 分配器（与各后端内存注册配合）：[`allocators.md`](allocators.md)
- 支持协议总览：[`../../docs/source/getting_started/supported-protocols.md`](../../docs/source/getting_started/supported-protocols.md)
- 官方设计目录：[`../../docs/source/design/transfer-engine/`](../../docs/source/design/transfer-engine/)
- 硬件协议矩阵（后端 × 分配器 × Store device 横向对照）：[`../00-overview/hardware-protocol-matrix.md`](../00-overview/hardware-protocol-matrix.md)
- 下一代 TENT 的 transport 子树（rdma/tcp/nvlink/shm/gds/io_uring/tpu/ascend/sunrise/fault_proxy）：[`tent.md`](tent.md)

## 关键源码入口

| 关注点 | 入口 |
|--------|------|
| 后端实例化工厂 | `mooncake-transfer-engine/src/multi_transport.cpp:314`（`installTransport`） |
| Transport 基类 | `mooncake-transfer-engine/include/transport/transport.h:44` |
| RDMA 主力后端 | `mooncake-transfer-engine/src/transport/rdma_transport/rdma_transport.cpp` |
| RDMA worker pool | `mooncake-transfer-engine/src/transport/rdma_transport/worker_pool.cpp` |
| RDMA endpoint 缓存 | `mooncake-transfer-engine/src/transport/rdma_transport/endpoint_store.cpp` |
| TCP 数据面 | `mooncake-transfer-engine/src/transport/tcp_transport/tcp_transport.cpp` |
| EFA libfabric | `mooncake-transfer-engine/src/transport/efa_transport/efa_transport.cpp` |
| CXI libfabric | `mooncake-transfer-engine/src/transport/cxi_transport/cxi_transport.cpp` |
| 跨节点 NVLink | `mooncake-transfer-engine/src/transport/nvlink_transport/nvlink_transport.cpp` |
| 节点内 NVLink | `mooncake-transfer-engine/src/transport/intranode_nvlink_transport/intranode_nvlink_transport.cpp` |
| GPUDirect Storage | `mooncake-transfer-engine/src/transport/nvmeof_transport/nvmeof_transport.cpp` |
| CXL | `mooncake-transfer-engine/src/transport/cxl_transport/cxl_transport.cpp` |
| Ascend direct | `mooncake-transfer-engine/src/transport/ascend_transport/ascend_direct_transport/` |
| HIP | `mooncake-transfer-engine/src/transport/hip_transport/hip_transport.cpp` |
| Sunrise/TPU | `mooncake-transfer-engine/src/transport/sunrise_link/sunrise_link_transport.cpp` |
| 昆鹏 UB | `mooncake-transfer-engine/src/transport/kunpeng_transport/ub_transport.cpp` |
| EP device transport | `mooncake-transfer-engine/src/transport/device/ibgda_device_transport.cpp` / `p2p_device_transport.cpp` |
