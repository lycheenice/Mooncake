# 异构硬件 × 传输协议/分配器 支持矩阵

本页汇总 Mooncake 对异构硬件、网络协议与内存分配器的横向支持，对应源码分布在 `mooncake-transfer-engine/` 与 `mooncake-store/src/device/` 两处。

## 一、传输协议（TE transport 后端）

源码：`mooncake-transfer-engine/src/transport/` 与 `include/transport/`。

| 协议 / 后端 | 目录 | 说明 |
|------------|------|------|
| RDMA (RoCE/IB) | `rdma_transport/` | 主力高性能后端，多 NIC 带宽聚合 |
| TCP | `tcp_transport/` | 通用回退，无 RDMA 环境 |
| NVLink | `nvlink_transport/` | 节点内 GPU-GPU |
| Intra-node NVLink | `intranode_nvlink_transport/` | 同进程 GPU 间 |
| AWS EFA | `efa_transport/` | AWS 自研互联 |
| NVMe-oF | `nvmeof_transport/` | 远端 SSD 块访问 |
| CXL | `cxl_transport/` | CXL.cache/mem |
| Barex | `barex_transport/` | 自研裸传输 |
| CXI | `cxi_transport/` | Cassini/Slingshot |
| RPC communicator | `rpc_communicator/` | 控制面信令 |
| Ascend | `ascend_transport/` | 华为 NPU（含 direct 变体） |
| HIP | `hip_transport/` | AMD GPU |
| MACA | `maca_transport/` | 寒武纪 MLU |
| Sunrise Link | `sunrise_link/` | Sunrise 互联 |
| Kunpeng UB | `kunpeng_transport/` | 昆鹏统一总线 |

官方支持协议文档：`../../docs/source/getting_started/supported-protocols.html`（docs 站中 `supported-protocols`）。

## 二、硬件加速器后端（Store device + TE allocator）

### TE 内存分配器

源码：`mooncake-transfer-engine/include/`。

| 分配器 | 头文件 | 适用硬件 |
|-------|--------|---------|
| cuda_alike | `cuda_alike.h` | NVIDIA / MUSA 等 CUDA-like |
| Ascend | `ascend_allocator.h` | 华为 NPU |
| Sunrise | `sunrise_allocator.h` | Sunrise |
| UB | `ub_allocator.h` | 昆鹏统一总线 |
| HIP guard | `hip_device_guard.h` | AMD HIP |
| GPU vendor | `gpu_vendor/` | 多厂商抽象 |

### Store 加速器设备

源码：`mooncake-store/src/device/`。

| 设备实现 | 文件 | 适用硬件 |
|---------|------|---------|
| CUDA-like | `cuda_like_accelerator_device.cpp` | NVIDIA / MUSA / MetaX / T-Head 等 CUDA-like |
| HIP | `hip_accelerator_device.cpp` | AMD |
| Ascend | `ascend_accelerator_device.cpp` | 华为 NPU |
| Sunrise | `sunrise_accelerator_device.cpp` | Sunrise |
| 运行时分发 | `runtime_accelerator.cpp` + `accelerator_registry.cpp` | 统一注册与选择 |

### 官方支持的硬件厂商

来自 `README.md`：NVIDIA · 华为 · AMD · 寒武弦纪 · 摩尔线程 · AWS · MetaX · 算能(T-Head) · 阿里云 · Sunrise · 海光。

## 三、SSD / 文件后端（Store 侧）

| 后端 | 源码 | 说明 |
|------|------|------|
| POSIX 文件 | `posix_file.cpp` | 通用 SSD |
| io_uring | `uring_file.cpp` | Linux 异步 I/O |
| SPDK | `spdk/` | NVMe 用户态 |
| 3FS (HF3FS) | `hf3fs/` | 3FS 文件系统 |
| CacheLib | `cachelib_memory_allocator/` | Meta CacheLib 内存层 |
| NVMe-oF (NoF) | `USE_NOF` 开关 | 远端 SSD pool |
| File storage | `file_storage.cpp` / `file_interface.h` | 统一文件接口 |
| mmap arena | `mmap_arena.cpp` | 共享内存大块 |

## 四、元数据 / 服务发现后端

| 后端 | 源码 / 开关 | 说明 |
|------|------------|------|
| etcd | `common/etcd` · `STORE_USE_ETCD` | 经典元数据 |
| Redis | `STORE_USE_REDIS` | KV 元数据 |
| HTTP metadata server | `http_metadata_server.cpp` | 自建 HTTP 元数据 |
| K8s Lease | `common/k8s-lease` · `STORE_USE_K8S_LEASE` | 云原生 leader election（与 etcd 互斥） |

## 五、官方矩阵参考

- docs 设计文档：`../../docs/source/design/transfer-engine/index.md`
- 各厂商 transport 专页：`../../docs/source/design/transfer-engine/`（含 `ascend_transport.md` / `efa_transport.md` / `kunpeng_ub_transport.md` / `sunrise_link_transport.md` / `ascend_direct_transport.md` / `heterogeneous_ascend.md`）
- 支持协议总览页：`../../docs/source/getting_started/supported-protocols.html`

---

> 注：各后端是否在当前构建中启用，取决于顶层 `CMakeLists.txt` 的 `WITH_*` / `USE_*` 开关与运行时环境检测。详见 [`dependency-graph.md`](dependency-graph.md) 的构建开关矩阵。
