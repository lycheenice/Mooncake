# Python 绑定层（mooncake-integration）

## 定位

`mooncake-integration` 是 C++ 内核（Transfer Engine / Store）与上层 Python 包之间的 **pybind11 桥梁**。它把 TE 的 `TransferEngine` 与 Store 的 `MooncakeDistributedStore` / `EngramStore` / `BufferPool` 包装成两个 Python 扩展模块 `engine` 与 `store`，并附带 fabric memory 分配器的 Python 封装。绑定机制依赖 git 子模块 `extern/pybind11`（顶层 `CMakeLists.txt:25`），由本目录的 `CMakeLists.txt` 调用 `pybind11_add_module` 构建出可在 CPython 中 `import` 的 `.so`。

integration 处于架构图的"绑定层"：其下是 C++ 内核（TE / Store / EP / PG），其上是 `mooncake-wheel` 的纯 Python 包，最终以 `pip install mooncake-transfer-engine` 形式交付外部生态（SGLang / vLLM / TRT-LLM …）。它是整栈 Python 链路的"装配现场"——`CMakeLists.txt` 不仅编译扩展模块，还把 `mooncake-wheel/mooncake/` 下的若干 `.py` 与 EP/PG 的 `.so` 一并 install 到 `site-packages/mooncake/`。

## 目录 / 源码结构

```
mooncake-integration/
├── CMakeLists.txt                    构建两个 pybind11 模块 + install Python 资产
├── integration_utils.h               共享头：Tensor 元数据协议 + dtype 桥接
├── transfer_engine/
│   ├── transfer_engine_py.h          TransferEnginePy 包装类声明
│   └── transfer_engine_py.cpp        PYBIND11_MODULE(engine) 实现
├── store/
│   ├── store_py.cpp                  PYBIND11_MODULE(store)：MooncakeDistributedStore 同步入口
│   ├── store_py_internal.h           Store 绑定实现库：并行读写路由 / 计划 / manifest 结构
│   ├── store_py_parallel_read.h      并行读实现（TP 分片 / 全对象重构路由）
│   ├── store_py_parallel_write.h     并行写实现（写者分区 / TP 分片 / 通用并行路由）
│   ├── engram_store_py.cpp           EngramStore（Conductor 索引）Python 绑定
│   ├── buffer_pool.cpp / .h          本地 buffer 租约池 BufferPool 绑定
│   └── async_store.py                MooncakeDistributedStoreAsync 异步包装
├── allocator.py                      NVLinkAllocator / BarexAllocator（MNNVL fabric memory）
├── allocator_ascend_npu.py           UBShmemAllocator（Ascend NPU UBSHMEM fabric memory）
└── fabric_allocator_utils.py         .so 定位 + fabric 后端能力探测
```

## 核心概念

### 1. 三层 Python 链路

```
C++ 内核 (mooncake-transfer-engine / mooncake-store / mooncake-ep / mooncake-pg)
    │  pybind11 (extern/pybind11, CMakeLists.txt:25)
    ▼
mooncake-integration/  (engine.so / store.so / ep.so / pg.so + allocator.py)
    │  import
    ▼
mooncake-wheel/mooncake/  (纯 Python 包：cli / store_service / connector / ep.py / pg.py)
    │  pip install mooncake-transfer-engine
    ▼
外部生态 (SGLang / vLLM / TRT-LLM …)
```

integration 同时承担"装配"职责：`CMakeLists.txt:160-168` 与 `:170-186` 把 `mooncake-wheel/mooncake/` 下的 `cli.py` / `mooncake_store_service.py` / `mooncake_config.py` / `http_metadata_server.py` / `mooncake_connector_v1.py` / `vllm_v1_proxy_server.py` 等 `.py` 连同编译产物一起 install 到 `site-packages/mooncake/`；`WITH_EP` 下再 install `ep.so` / `pg.so` 并创建 `engine.so` → `engine<EXT_SUFFIX>` 符号链接（`CMakeLists.txt:207-228`）。wheel 的 pip 打包（`setup.py`）是另一套机制，二者关系见 [`../09-wheel/index.md`](../09-wheel/index.md)。

### 2. 两个 pybind11 模块

- **`engine`**（`WITH_TE`，`CMakeLists.txt:46`）：源含 `transfer_engine/transfer_engine_py.cpp`，链接 `transfer_engine` + `rpc_communicator`，按 `USE_EFA / USE_CXI / USE_TENT / USE_CUDA / USE_ASCEND_DIRECT` 传播编译定义。产出 `engine<EXT_SUFFIX>.so`。
- **`store`**（`WITH_STORE`，`CMakeLists.txt:107`）：源含 `store/store_py.cpp` + `store/buffer_pool.cpp` + `store/engram_store_py.cpp`，链接 `mooncake_store` + `cachelib_memory_allocator`。产出 `store<EXT_SUFFIX>.so`。

两者 `INSTALL_RPATH` 设为 `$ORIGIN`（`CMakeLists.txt:49,115`），确保从任意目录 import 时能找到同目录下的 `libtransfer_engine.so` 等依赖。

### 3. Tensor 元数据协议

`integration_utils.h` 定义了 Store put/get tensor 的**线规**，是 Store 与并行分片层共享的契约：

- `kTensorObjectMagic = 0x4d4f4f4e`（"MOON"，`integration_utils.h:171`）+ `kTensorObjectVersion = 1`。
- `TensorObjectHeader`（`integration_utils.h:211`）：magic / version / header_size / dtype / ndim / layout_kind / data_offset / data_bytes。
- `TensorLayoutMetadata`（`integration_utils.h:203`）：`global_shape` + `local_shape`（最多 `kMaxTensorDims=8` 维）+ 最多 `kMaxLayoutAxes=4` 条 `LayoutAxis`，axis kind 取 `DP / TP / EP / PP / RESERVED`（`integration_utils.h:181`）。
- `ValidateTensorMetadata`（`integration_utils.h:308`）做完整一致性校验（magic / dtype / ndim / data_bytes 与 local_shape 乘积一致 / 未用维度填 -1 / axis shard 合法性）；`ParseTensorMetadata`（`:383`）从裸字节切出 header + data。

这层线规使得 `put_tensor_with_tp` / `get_tensor_with_parallelism` 等分片 API 能在 writer/reader 两侧无损还原全局 shape 与分片编排，是 [`../03-store/index.md`](../03-store/index.md) 中 Tensor-Centric 存储的 Python 侧落地。

### 4. PyTorch dtype 桥接

`integration_utils.h` 用 `torch_module()` 懒加载 `torch`，提供 15 种 dtype 的双向映射：`get_tensor_dtype`（`:105`，`torch.float32` → `TensorDtype::FLOAT32`…，含 `float8_e4m3fn / float8_e5m2`）与 `tensor_dtype_to_torch_dtype`（`:133`）。`array_creators`（`:61`）按 dtype 把裸 buffer 包成 `py::array_t<T>`，支持 `take_ownership`（capsule 释放）与外部托管两种模式。这是 `get_tensor` 返回 numpy/torch 视图、零拷贝暴露 Segment 内存的实现基础。

### 5. Store API 的同步 / 异步 / 并行三档

- **同步**：`MooncakeStorePyWrapper`（`store_py.cpp:359`）封装 `RealClient` / `DummyClient`，绑定名 `MooncakeDistributedStore`（`store_py.cpp:2100`）。暴露 `put / get / put_tensor / get_tensor / put_tensor_with_tp / batch_* / pub_tensor / upsert_tensor / get_tensor_into / put_tensor_from / save_tensor_to_safetensor` 等数十个方法（绑定见 `store_py.cpp:2197` 起）。
- **异步**：`MooncakeDistributedStoreAsync`（`async_store.py:5`）继承同步类，用 `__getattr__` 拦截 `async_*` 调用，把同名同步方法经 `loop.run_in_executor(None, func)` 包成协程，实现零样板的异步 API。
- **并行**：`store_py_internal.h` 定义读写路由枚举——`ParallelismShardReadStorageRoute`（`:118`，按 key / 远端分片等路由重构全张量）与 `ParallelismWriteStorageRoute`（`:666`，含 `DIRECT_FULL_OBJECT / WRITER_PARTITION_SHARD / LEGACY_SINGLE_TP / TP_SHARDED_PARALLELISM / GENERIC_PARALLELISM_SHARD`）。`store_py_parallel_read.h` / `store_py_parallel_write.h` 据此用多线程把一个逻辑张量的分片并行下发到 Store，对应 `get_tensor_with_parallelism` / `put_tensor_with_parallelism` 系列（绑定 `store_py.cpp:2479` 起）。

### 6. 本地 BufferPool

`bind_buffer_pool`（`buffer_pool.cpp:536`）暴露 `BufferPoolNative`（`buffer_pool.cpp:127`）+ `BufferLeaseNative`（`:94`）+ `BufferLeaseViewNative`（`:78`）。`acquire(size)` / `buffer(size)` 返回 lease，支持 `with` 语法（`__enter__/__exit__`，`:550`），`prewarm(size, count)` 预热 size class，`close()` 释放池。这是本地 DRAM 上按 size class 分桶的租约池，避免高频 put/get 反复注册内存，被 wheel 侧 `buffer_pool.py` 转引暴露为 `BufferPool`。

### 7. Engram / Conductor 绑定

`bind_engram_store`（`engram_store_py.cpp:172`）暴露 `EngramStoreConfig` 与 `EngramStore`，方法含 `get_table_vocab_sizes / get_store_keys / get_num_heads / get_embedding_dim`。`EngramStore` 即 docs 中 Conductor 索引的实现（参见 [`../03-store/engram-conductor.md`](../03-store/engram-conductor.md)），此绑定把"对象 → 位置"智能索引交给 Python 侧使用。

### 8. Fabric memory 分配器

`allocator.py` 与 `allocator_ascend_npu.py` 把 GPU/NPU 显存分配接到 Mooncake 的 fabric memory（跨节点统一寻址）：

- `NVLinkAllocator`（`allocator.py:24`）：基于 `torch.cuda.memory.CUDAPluggableAllocator` + `nvlink_allocator.so`（MNNVL），`detect_mem_backend` 探测 `USE_CUDAMALLOC / USE_CUMEMCREATE`，按 device 缓存 allocator 实例。
- `UBShmemAllocator`（`allocator_ascend_npu.py:14`）：基于 `torch_npu.npu.memory.NPUPluggableAllocator` + `ubshmem_fabric_allocator.so`（USE_UBSHMEM），探测 `aclMallocPhysical` 能力。
- `BarexAllocator`（`allocator.py:92`）：定位 `libaccl_barex.so`，走 `u2mm_alloc/free_wrapper_with_stream`。
- `fabric_allocator_utils.py` 的 `get_mooncake_so_path`（`:12`）定位包内 `.so`，`probe_allocator_backend`（`:33`）用 ctypes 调 `mc_allocator_probe` 探测后端。

这些 `.so` 与 `.py` 由 integration `CMakeLists.txt` 按 `USE_MNNVL / USE_UBSHMEM` 条件 install（`:88-150`），仅在对应 fabric 硬件构建下出现在 `site-packages/mooncake/`。注意 `allocator_ascend_npu.py` 被 `.gitignore` 忽略（由构建期生成/拷入）。

## 交叉引用

- 传输引擎内核（`engine` 模块的底层）：[`../02-transfer-engine/index.md`](../02-transfer-engine/index.md)
- 存储引擎内核（`store` 模块的底层 + Engram/Conductor）：[`../03-store/index.md`](../03-store/index.md)
- 打包分发层（import integration 产物的 wheel 包）：[`../09-wheel/index.md`](../09-wheel/index.md)
- EP / PG 绑定安装（`WITH_EP` 下 .so install）：[`../05-ep/index.md`](../05-ep/index.md) · [`../06-pg/index.md`](../06-pg/index.md)
- Python 全链路三层递进图：[`../00-overview/dependency-graph.md`](../00-overview/dependency-graph.md)（第 6 节）
- 架构拓扑与 integration 定位：[`../00-overview/architecture.md`](../00-overview/architecture.md)
- 官方 Python API 参考：[`../../docs/source/api-reference/python/index.md`](../../docs/source/api-reference/python/index.md)

## 关键源码入口

| 关注点 | 入口 |
|--------|------|
| pybind11 子模块接入 | `CMakeLists.txt:25`（`add_subdirectory(extern/pybind11)`） |
| `engine` 模块构建 | `mooncake-integration/CMakeLists.txt:46`（`pybind11_add_module(engine ...)`） |
| `store` 模块构建 | `mooncake-integration/CMakeLists.txt:107`（`pybind11_add_module(store ...)`） |
| wheel `.py` 装配 install | `mooncake-integration/CMakeLists.txt:160`（TE 侧）/ `:171`（Store 侧）/ `:178`（NOF） |
| EP/PG `.so` install + 符号链接 | `mooncake-integration/CMakeLists.txt:207`（WITH_EP） |
| `engine` 模块绑定 | `mooncake-integration/transfer_engine/transfer_engine_py.cpp:1118`（`PYBIND11_MODULE(engine)`） |
| TE Python 包装类 | `mooncake-integration/transfer_engine/transfer_engine_py.h:43`（`TransferEnginePy`） |
| `store` 模块绑定 | `mooncake-integration/store/store_py.cpp:1806`（`PYBIND11_MODULE(store)`） |
| Store 同步包装类 | `mooncake-integration/store/store_py.cpp:359`（`MooncakeStorePyWrapper`）/ `:2100`（绑定 `MooncakeDistributedStore`） |
| 异步包装 | `mooncake-integration/store/async_store.py:5`（`MooncakeDistributedStoreAsync`） |
| 并行读路由 | `mooncake-integration/store/store_py_internal.h:118`（`ParallelismShardReadStorageRoute`） |
| 并行写路由 | `mooncake-integration/store/store_py_internal.h:666`（`ParallelismWriteStorageRoute`） |
| EngramStore 绑定 | `mooncake-integration/store/engram_store_py.cpp:172`（`bind_engram_store`） |
| BufferPool 绑定 | `mooncake-integration/store/buffer_pool.cpp:536`（`bind_buffer_pool`）/ `:127`（`BufferPoolNative`） |
| Tensor 元数据 magic | `mooncake-integration/integration_utils.h:171`（`kTensorObjectMagic`）/ `:211`（`TensorObjectHeader`） |
| Tensor 元数据校验 | `mooncake-integration/integration_utils.h:308`（`ValidateTensorMetadata`）/ `:383`（`ParseTensorMetadata`） |
| dtype 桥接 | `mooncake-integration/integration_utils.h:105`（`get_tensor_dtype`）/ `:133`（`tensor_dtype_to_torch_dtype`） |
| NVLink fabric allocator | `mooncake-integration/allocator.py:24`（`NVLinkAllocator`）/ `:92`（`BarexAllocator`） |
| UBShmem fabric allocator | `mooncake-integration/allocator_ascend_npu.py:14`（`UBShmemAllocator`） |
| .so 定位与后端探测 | `mooncake-integration/fabric_allocator_utils.py:12`（`get_mooncake_so_path`）/ `:33`（`probe_allocator_backend`） |
