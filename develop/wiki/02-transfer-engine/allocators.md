# 内存分配器（Memory Allocators）

## 定位

`mooncake-transfer-engine/include/` 下的分配器头为异构硬件（GPU/NPU/UB 共享内存）提供统一的内存分配与释放接口，并承担**让分配出的内存可被 RDMA 等 transport 零拷贝注册**的职责。它们不是通用 malloc 替代，而是面向 zero-copy 数据路径的厂商适配层：分配时常驻于设备显存或 pinned host memory，并附带物理句柄供 transport 注册 MR。

分配器与 transport 后端（见 [`transports.md`](transports.md)）成对出现：同一硬件家族的 transport 负责搬移，allocator 负责把内存按该硬件可注册的方式分配出来。上层 Store 也复用同一批分配器语义（`Store/device/accelerator_*`）。

## 目录 / 源码结构

```
mooncake-transfer-engine/include/
├── cuda_alike.h          编译期分派入口：按 USE_* 选 gpu_vendor/ 头
├── ascend_allocator.h    华为 Ascend NPU（ACL VMM）
├── sunrise_allocator.h   Sunrise/TPU（tang runtime，含 store range 跟踪）
├── ub_allocator.h        昆鹏 UB（Urma）
├── hip_device_guard.h    AMD HIP device RAII 守卫
└── gpu_vendor/           各厂商头 shim（设 GPU_PREFIX + CU→厂商宏映射）
    ├── hip.h             AMD ROCm
    ├── musa.h            摩尔线程 MUSA
    ├── mlu.h             寒武纪 MLU
    ├── maca.h            寒武纪 MC runtime（CUDA-like 面）
    ├── mnnvl.h           NVIDIA MNNVL fabric handle 桥接宏
    ├── ubshmem.h         Ascend UB 共享内存（GPU_PREFIX="npu:"）
    ├── sunrise.h         Sunrise/TPU tang runtime
    └── intra_nvlink.h    节点内 NVLink shim
```

> 注意：`ascend_allocator` 的实现位于 `src/transport/ascend_transport/ascend_allocator.cpp`；`ub_allocator` 实现在 `src/transport/kunpeng_transport/ub_allocator.cpp`。`sunrise_allocator.h` 是 header-only inline 实现。`cuda_alike.h` 本身不分配内存，只做 API 统一。

## 核心概念

### 1. `cuda_alike.h` —— 编译期厂商分派

`cuda_alike.h` 是整个异构内存抽象的**枢纽**。它通过一串 `#ifdef USE_CUDA / USE_HIP / USE_MUSA / USE_MLU / USE_UBSHMEM / USE_MACA / USE_SUNRISE / USE_HYGON / USE_COREX` 条件，include 对应 `gpu_vendor/*.h`，从而把各家厂商的运行时 API 统一成 CUDA-like 调用面（`cuMemCreate` / `cuMemExportToShareableHandle` / `tangMalloc` / `aclrtMalloc` …）。

各 `gpu_vendor/*.h` 主要做两件事：

- 设 `GPU_PREFIX`（`"cuda:"` / `"hip:"` / `"maca:"` / `"npu:"` 等），供 transport 与 metadata 识别内存 location 类型。
- 用宏把 CUDA 符号映射到厂商符号（如 `maca.h`：`CUdevice→MCdevice`、`cuMemCreate→mcMemCreate`；`hip.h`：`CU_MEM_HANDLE_TYPE_FABRIC→hipMemHandleTypePosixFileDescriptor`）。

这样上层 transport / Store 代码可用一套 CUDA-like 写法覆盖多厂商，差异被收拢在 `gpu_vendor/` 头里。

### 2. `ascend_allocator.h` —— Ascend NPU

适用硬件：华为 Ascend NPU（`USE_ASCEND_DIRECT` / `USE_UBSHMEM`）。

接口：

- `ascend_allocate_memory(size, protocol)` —— 按 protocol（RoCE / HCCS / UB shm）选分配策略，走 ACL `aclrtMalloc` 或 VMM。
- `ascend_allocate_vmm_memory_direct(size)` —— 直接 ACL VMM 分配，绕过 adxl `MallocMem`，供 `ascend_agent_mode && ascend_use_fabric_mem` 的 `shm_helper` 使用；要求 size 为 1GB 倍数。
- `aclrtDrvMemHandle ascend_get_physical_handle_from_va(va)` —— 从虚地址反查物理句柄，供 RDMA MR 注册做 zero-copy。
- `ascend_is_store_memory(addr, length)` —— 判断是否与 Store 内存区间重叠，决定 RoCE+store 注册策略。

与 transport 配合：`ascend_direct_transport` 用物理句柄注册 RoCE MR；`hccl_transport` 走 HCCS；`ubshmem_transport` 走 UB 共享内存。`GlobalConfig.ascend_use_fabric_mem` / `ascend_agent_mode` / `ascend_store_te_init`（`config.h:111-120`）控制其行为。

### 3. `sunrise_allocator.h` —— Sunrise/TPU

适用硬件：Sunrise/TPU（`USE_SUNRISE`，tang runtime）。

特点：

- header-only inline，无独立 `.cpp`。
- 两条分配路径：`use_device_mem=true` → `tangMalloc`（设备显存）；否则 → `tangHostAlloc`（pinned host），失败回退 `posix_memalign`。
- 维护两个全局集合（`tangDeviceAllocatedSet` / `tangHostAllocatedSet`）+ 跟踪 store 内存 range，供 `sunrise_is_device_memory_range` / `sunrise_is_host_allocated` 反查类型，从而决定 transport 选 device 直连还是 host 路径。
- `GlobalConfig.sunrise_use_device_mem`（`config.h:113`）开关键。

与 transport 配合：`sunrise_link_transport` 依 `sunrise_is_device_memory_range` 选路径，设备内存走 TPU 直连，host 内存走 RDMA/TCP。

### 4. `ub_allocator.h` —— 昆鹏 UB/Urma

适用硬件：海光昆鹏 UB（`USE_UB`，Urma jfc/jfce/jetty）。

接口：

- `ub_allocate_memory(alignment, total_size)` / `ub_free_memory(ptr)` —— UB 共享内存分配释放。
- `ub_is_store_memory(addr, length)` —— 与 ascend/sunrise 同构的 store 区间判定。

实现见 `src/transport/kunpeng_transport/ub_allocator.cpp`。与 `ub_transport` 配合：UB 段在 `BufferDesc.tseg/l_seg_index`（`transfer_metadata.h:75-76`）携带 segment 信息，`HandShakeDesc.jetty_num` 携带 jetty 号，allocator 负责分配底层可被 jfc/jfce 注册的内存。

### 5. `hip_device_guard.h` —— HIP device 守卫

适用硬件：AMD ROCm（`USE_HIP` / `USE_HIP_DMABUF`）。

这不是分配器，而是 RAII 守卫：构造时保存当前 `hipGetDevice`，可切换到 target device，析构时 `hipSetDevice` 还原。防止某次内存注册/cuFile 操作切走 device 后，调用方后续 kernel launch 因 device 不符而失败。`hip_transport` 与 `intranode_nvlink_transport` 的 HIP 路径在操作前用此守卫保护。

### 6. 内存注册与 zero-copy 链路

分配器分配 → `TransferEngine::registerLocalMemory(addr, length, location)`（`index.md` 入口）→ transport 的 `install` 把 buffer 注册为 MR（RDMA：`ibv_reg_mr`；HIP：IPC handle 导出；NVLink：fabric handle 导出；Ascend：物理句柄→MR）→ `TransferMetadata::addLocalMemoryBuffer` 写入 `BufferDesc`（含 lkey/rkey/shm_name/offset/tseg）→ 握手广播给 peer → peer 即可零拷贝 R/W。

`location` 字符串（如 `cpu:0` / `gpu:0` / `npu:0` / `segments:4096:1,3`）由 `memory_location.h` 体系解析，决定拓扑选 NIC（见 [`index.md`](index.md) 拓扑感知一节）。

## 交叉引用

- 引擎核心与 `registerLocalMemory` 链路：[`index.md`](index.md)
- 各 transport 后端如何消费分配出的内存：[`transports.md`](transports.md)
- 硬件协议矩阵（allocator × transport × Store device 横向对照）：[`../00-overview/hardware-protocol-matrix.md`](../00-overview/hardware-protocol-matrix.md)
- 官方异构 Ascend 文档：[`../../docs/source/design/transfer-engine/heterogeneous_ascend.md`](../../docs/source/design/transfer-engine/heterogeneous_ascend.md)
- TENT platform 层（同语义的 per-vendor allocator/probe，为 TENT 重写）：[`tent.md`](tent.md)

## 关键源码入口

| 关注点 | 入口 |
|--------|------|
| 厂商编译期分派 | `mooncake-transfer-engine/include/cuda_alike.h:1` |
| HIP 厂商 shim | `mooncake-transfer-engine/include/gpu_vendor/hip.h:1` |
| MACA 厂商 shim | `mooncake-transfer-engine/include/gpu_vendor/maca.h:1` |
| MNNVL fabric handle 桥接 | `mooncake-transfer-engine/include/gpu_vendor/mnnvl.h:1` |
| Ascend 分配器接口 | `mooncake-transfer-engine/include/ascend_allocator.h:9` |
| Ascend 分配器实现 | `mooncake-transfer-engine/src/transport/ascend_transport/ascend_allocator.cpp` |
| Sunrise 分配器（inline） | `mooncake-transfer-engine/include/sunrise_allocator.h:66`（`sunrise_allocate_memory`） |
| UB 分配器接口 | `mooncake-transfer-engine/include/ub_allocator.h:4` |
| UB 分配器实现 | `mooncake-transfer-engine/src/transport/kunpeng_transport/ub_allocator.cpp` |
| HIP device 守卫 | `mooncake-transfer-engine/include/hip_device_guard.h:26`（`HipDeviceGuard`） |
| location 编码解析 | `mooncake-transfer-engine/include/memory_location.h:57`（`SegmentsLocationInfo`） |
| 内存注册入口 | `mooncake-transfer-engine/src/transfer_engine_impl.cpp:586`（`registerLocalMemory`） |
| BufferDesc 字段 | `mooncake-transfer-engine/include/transfer_metadata.h:56` |
