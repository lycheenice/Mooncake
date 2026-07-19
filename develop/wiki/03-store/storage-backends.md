# Store 多级存储后端

## 定位

`mooncake-store` 把 worker 节点上的异构存储介质组织成四级层次：**VRAM → DRAM → SSD → 远端共享存储**。每一级对应一类 `BufferAllocator` 与一组存储后端实现，上层的 `allocation_strategy` 在这些 Segment 之间做放置决策，`storage_backend` 负责本地 SSD 的 bucket 化落盘（offload），`device/` 则把加速器显存注册进 TE。本页梳理这四级的源码结构和选用开关。

## 目录 / 源码结构

```
mooncake-store/src/
├── device/                              VRAM（加速器显存）层
│   ├── accelerator_device.cpp           AcceleratorDevice 抽象基类
│   ├── accelerator_registry.cpp         静态注册表（启动期探测可用 vendor）
│   ├── runtime_accelerator.cpp          RuntimeAccelerator：运行期 H2D/D2H 拷贝
│   ├── cuda_like_accelerator_device.cpp NVIDIA / MUSA / Corex / Hygon 等同构 CUDA-like 设备
│   ├── hip_accelerator_device.cpp       AMD HIP
│   ├── ascend_accelerator_device.cpp    华为 Ascend NPU
│   └── sunrise_accelerator_device.cpp   Sunrise
├── cachelib_memory_allocator/           DRAM 层：Facebook cachelib 移植（Pool/Slab 分配）
├── memory_alloc.cpp                     hugepage_memory_alloc/free
├── mmap_arena.cpp                       MmapArena：lock-free CAS arena（SGLang HiCache 复用）
├── allocator.cpp                        CachelibBufferAllocator / OffsetBufferAllocator 包装
├── offset_allocator/                    段内偏移分配器（bitmap + first-fit）
├── file_storage.cpp                     本地 SSD bucket 化存储（offload 落盘主体）
├── posix_file.cpp                       StorageFile 的 POSIX read/write/pread 实现
├── uring_file.cpp                       StorageFile 的 io_uring 实现（per-thread ring）
├── spdk/                                SPDK NVMe 用户态 polled-mode（USE_NOF）
│   ├── spdk_wrapper.cpp                 SpdkWrapper 单例（env init / DMA buf）
│   └── CMakeLists.txt
├── hf3fs/                               3FS（DeepSeek 3FS）远端文件系统适配
│   ├── hf3fs_file.cpp                   HF3FS StorageFile 实现
│   ├── hf3fs_resource_manager.cpp
│   └── README.md
├── storage/distributed/                 分布式共享 Segment 后端
│   ├── distributed_storage_backend.cpp  DistributedStorageBackend 主体
│   └── hf3fs_adapter.cpp                3FS 适配器
├── ssd_register_client.cpp              NoFRegisterClient：向 master 注册/反注册 NVMe-oF Segment
└── pinned_buffer_pool.h / pinned_host_buffer.h   pinned host buffer 池（D2H 加速）

对应头文件：include/device/ · include/storage/distributed/ · include/hf3fs/ · include/spdk/
    include/cachelib_memory_allocator/ · include/file_interface.h · include/storage_backend.h
    include/offset_allocator/ · include/mmap_arena.h · include/memory_alloc.h · include/allocator.h
    include/ssd_register_client.h · include/pinned_*.h
```

## 核心概念

### 四级存储层次

| 层级 | 介质 | 代表实现 / 文件 | 角色 | 关键开关 |
|------|------|----------------|------|---------|
| VRAM | 加速器显存 | `device/cuda_like_accelerator_device.cpp` / `hip_*` / `ascend_*` / `sunrise_*` | 通过 `AcceleratorDevice::AllocatePinnedHost` + TE 注册显存段，供 GPU 直读 | vendor 自动探测 |
| DRAM | 主机内存 | `cachelib_memory_allocator/` · `mmap_arena.cpp` · `memory_alloc.cpp` | 主对象池，`OffsetBufferAllocator` 或 `CachelibBufferAllocator` 分配 | `FLAGS_memory_allocator={offset,cachelib}` |
| SSD | 本地 NVMe | `file_storage.cpp` + `posix_file.cpp` / `uring_file.cpp`；`spdk/spdk_wrapper.cpp` | DRAM 满时 offload 落盘（`LOCAL_DISK` replica），band-bucket 化存储 | `USE_URING`、`USE_NOF` |
| 远端 | 共享存储 | `storage/distributed/distributed_storage_backend.cpp` + `hf3fs/`；`ssd_register_client.cpp` | 跨节点共享 Segment（3FS）或 NVMe-oF（NoF） | `USE_NOF`、3FS lib |

### 加速器 device 抽象

`AcceleratorDevice`（`device/accelerator_device.h:42`）定义跨 vendor 接口：`QueryPointer`（判断指针是 host/device）、`Copy`（H2D / D2H / D2D）、`AllocatePinnedHost`（pinned host buffer，D2H 带宽 10x~100x 于 pageable）。`AcceleratorRegistry` 在启动期枚举已编译的 vendor 实现，`RuntimeAccelerator`（`runtime_accelerator.h:12`）聚合它们，提供指针→设备查找与统一 `CopyToHost` / `CopyFromHost`。这与 `02-transfer-engine` 的 transport/allocator 是**同一批硬件的存储侧镜像**（见 [`../00-overview/hardware-protocol-matrix.md`](../00-overview/hardware-protocol-matrix.md)）。

### DRAM 分配器三选

- `OffsetBufferAllocator`：基于 `offset_allocator/` 的偏移分配（默认），轻量、快。
- `CachelibBufferAllocator`：Facebook cachelib 的 Pool/Slab 分配，支持更细粒度回收，用于 `FLAGS_memory_allocator=cachelib`。
- `MmapArena`（`mmap_arena.h:19`）：lock-free CAS bump allocator，给 SGLang HiCache 等外部消费者快速切分大页 pool。

`AllocatorManager`（`allocation_strategy.h:26`）按 Segment 名管理多分配器实例，由 `SegmentManager` 在 `segment_mutex_` 下访问。

### 分配策略

`FLAGS_allocation_strategy` 决定新 replica 放在哪个 Segment，可选 `random` / `free_ratio_first` / `cxl` / `ssd_free_ratio_first` / `local_first`（`master.cpp:289`）。`free_ratio_first` 与 `ssd_free_ratio_first` 按"空闲比例"优先排序，后者把 SSD 段也纳入候选——对应官方 `ssd-free-ratio-first-allocation` 设计。`cxl` 策略配合 `FLAGS_enable_cxl` 与 `MC_CXL_DEV_SIZE` 把 Segment 落到 CXL DAX 设备。

### 本地 SSD offload（bucket 化）

`StorageBackend`（`storage_backend.h`，`src/storage_backend.cpp`）把本地 SSD 切成多个 bucket，每个 `BucketMetadata` 记录 `keys` + `BucketObjectMetadata`（offset/key_size/data_size），用 `struct_pack` 序列化。读路径用 `BucketReadGuard`（`storage_backend.h:108`）RAII 计 `inflight_reads_`，保证 bucket 删除时等所有在飞读完成。文件 I/O 经 `file_interface.h` 的 `StorageFile` 抽象，落 `PosixFile`（`posix_file.cpp`）或 `UringFile`（`uring_file.cpp`，per-thread ring + 全局 buffer 注册）。offload 由 master 心跳驱动（`OffloadObjectHeartbeat` → `NotifyOffloadSuccess`），promotion 反向（`PromotionObjectHeartbeat` → `PromotionAllocStart` → `NotifyPromotionSuccess`）。

### SPDK / NVMe-oF（NoF）

`USE_NOF` 开关打开后：client 侧 `SpdkWrapper::GetInstance().InitializeEnv()` 初始化 SPDK 环境，client buffer 走 SPDK DMA；master 侧通过 `NoFRegisterClient`（`ssd_register_client.h:12`）的 `set_register` / `set_unregister_by_endpoint` 把 NVMe-oF Segment 注册为 `NOF_SSD` replica。master 周期性心跳探测 NoF segment（`FLAGS_nof_heartbeat_*`），连续 `nof_heartbeat_failures_threshold` 次失败则 unmount。NoF 走独立驱逐路径（`NoFBatchEvict` / `nof_eviction_ratio`）。

### 远端共享存储：3FS 与 DistributedStorageBackend

`storage/distributed/distributed_storage_backend.cpp` + `hf3fs_adapter.cpp` 把 DeepSeek 3FS（`hf3fs/`）挂成全局共享 Segment（`FLAGS_global_file_segment_size`），支持跨节点读写同一 bucket，配合 `FLAGS_root_fs_dir` 与 `FLAGS_cluster_id` 做 kvcache 持久化（HA 模式下使用）。

## 交叉引用

- SSD offload 设计：[`../../docs/source/design/ssd-offload.md`](../../docs/source/design/ssd-offload.md)
- free-ratio-first 分配设计：[`../../docs/source/design/ssd-free-ratio-first-allocation.md`](../../docs/source/design/ssd-free-ratio-first-allocation.md)
- 异构硬件横向矩阵：[`../00-overview/hardware-protocol-matrix.md`](../00-overview/hardware-protocol-matrix.md)
- Store 总览：[`index.md`](index.md)
- 元数据 / HA（snapshot payload/catalog 后端选择）：[`metadata-ha.md`](metadata-ha.md)

## 关键源码入口

- `mooncake-store/include/device/accelerator_device.h:42` — `class AcceleratorDevice` 跨 vendor 接口
- `mooncake-store/include/device/runtime_accelerator.h:12` — `RuntimeAccelerator` 聚合器
- `mooncake-store/src/device/accelerator_registry.cpp` — 启动期 vendor 注册表
- `mooncake-store/src/device/cuda_like_accelerator_device.cpp` — CUDA-like 显存实现
- `mooncake-store/include/allocator.h:21` — `enum ReplicaType`（MEMORY/DISK/LOCAL_DISK/NOF_SSD）
- `mooncake-store/include/allocator.h:36` — `AllocatedBuffer` buffer 句柄
- `mooncake-store/include/allocation_strategy.h:26` — `AllocatorManager` Segment 内分配器容器
- `mooncake-store/include/mmap_arena.h:19` — `class MmapArena` lock-free arena
- `mooncake-store/include/memory_alloc.h:6` — `hugepage_memory_alloc/free`
- `mooncake-store/include/storage_backend.h:96` — `class BucketReadGuard` RAII 读计数
- `mooncake-store/include/file_interface.h:57` — `class StorageFile` 文件 I/O 抽象
- `mooncake-store/src/posix_file.cpp:10` — `PosixFile` 实现
- `mooncake-store/src/uring_file.cpp:1` — `UringFile`（`#ifdef USE_URING`，per-thread ring）
- `mooncake-store/src/spdk/spdk_wrapper.cpp` — `SpdkWrapper` SPDK 环境单例
- `mooncake-store/include/ssd_register_client.h:12` — `class NoFRegisterClient` NoF Segment 注册
- `mooncake-store/src/storage/distributed/distributed_storage_backend.cpp` — `DistributedStorageBackend` 远端共享 Segment
- `mooncake-store/src/hf3fs/hf3fs_file.cpp` — 3FS `StorageFile` 实现
- `mooncake-store/src/storage_backend.cpp` — 本地 SSD bucket 化 offload 主体
- `mooncake-store/include/pinned_buffer_pool.h:25` — `PinnedBufferPool` pinned host buffer 池
- `mooncake-store/include/pinned_host_buffer.h:10` — `PinnedHostBuffer` RAII 句柄
- `CMakeLists.txt:60` — `USE_NOF` 开关（root `CMakeLists.txt:60-64`）
