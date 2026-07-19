# Go / Rust 绑定

## 一、定位

本页覆盖 `mooncake-store` 为非 C++ 语言提供的两类本机绑定：**Go 绑定**（CGo，`WITH_STORE_GO`）与 **Rust 绑定**（bindgen，`WITH_STORE_RUST`）。二者均包装同一份 C API `store_c.h`，使 Go / Rust 程序能直接以 RDMA/TCP 高性能读写 Mooncake Store。Python 绑定不在此层（见 [`../09-wheel/index.md`](../09-wheel/index.md) 与 [`../08-integration/index.md`](../08-integration/index.md)）。

## 二、目录 / 源码结构

```
mooncake-store/
├── include/store_c.h                二者共用的 C API 契约 (opaque handle + config)
├── src/store_c.cpp                  C API 实现 (转发到 Client)
├── go/
│   ├── go.mod                       module github.com/kvcache-ai/Mooncake/mooncake-store/go
│   ├── build.sh                     CGo 构建：设 CGO_CFLAGS/CGO_LDFLAGS 并 go build
│   ├── mooncakestore/
│   │   ├── store.go                 Store 类型 + CGo 包装 (Put/Get/Remove/Batch/Register…)
│   │   ├── config.go                ReplicateConfig / DefaultReplicateConfig
│   │   └── errors.go                错误常量
│   ├── examples/basic/              示例二进制 (build.sh 产出 mooncake-store-go-example)
│   └── tests/integration_test.go    集成测试
└── rust/
    ├── README.md                    构建/运行说明 (CMake 与 Cargo 两种路径)
    ├── Cargo.toml                   crate mooncake_store (rlib, bindgen build-dep)
    ├── Cargo.lock
    ├── build.rs                     bindgen 从 store_c.h 生成 FFI bindings.rs
    ├── CMakeLists.txt               三个 custom target：crate / examples / tests
    ├── src/
    │   ├── lib.rs                   模块导出 MooncakeStore / ReplicateConfig / StoreError
    │   ├── store.rs                 safe 包装 + 生命周期 + Send/Sync
    │   └── error.rs                 StoreError 枚举 (thiserror)
    ├── examples/                    basic_usage.rs · store_benchmark.rs
    └── tests/minimal_smoke.rs       集成冒烟测试
```

## 三、核心概念

### 1. 共用 C API (`store_c.h`)

Go 与 Rust 都包装 `store_c.h`（`store_c.h:25`）。该头文件以 `typedef void *mooncake_store_t` 不透明句柄 + `mooncake_replicate_config_t`（`replica_num` / `with_soft_pin` / `with_hard_pin` / `preferred_segments` 数组）为契约，导出全量接口：生命周期（`create`/`destroy`/`setup`/`init_all`/`health_check`）、写入（`put`/`put_from`/`batch_put_from`）、读取（`get_into`/`batch_get_into`）、存在性/尺寸（`is_exist`/`batch_is_exist`/`get_size`/`get_hostname`）、删除（`remove`/`remove_by_regex`/`remove_all`）、零拷贝缓冲注册（`register_buffer`/`unregister_buffer`）。约定所有 `char *` 入参在函数返回后不再使用，调用方可立即释放。

### 2. 构建开关与依赖

| CMake option | 默认 | 约束 | 入口 |
|--------------|------|------|------|
| `WITH_STORE_GO` | OFF | 需 `WITH_STORE=ON` | `CMakeLists.txt:17` · `CMakeLists.txt:181` |
| `WITH_STORE_RUST` | ON | 需 `WITH_STORE=ON`（`CMakeLists.txt:89`） | `CMakeLists.txt:20` · `CMakeLists.txt:92` |

Go 经 `build.sh`（`go/build.sh`）手工设置 `CGO_CFLAGS`（指向 `store_c.h` 与 TE 头）与 `CGO_LDFLAGS`（链接 `libmooncake_store` / `libtransfer_engine` / `libcachelib_memory_allocator` / `libmooncake_common` 及 glog/zstd/numa/ibverbs/jsoncpp 等；可选 etcd/redis/libzmq），CMake 自定义目标 `build_store_go` 调用之。Rust 由 `rust/CMakeLists.txt` 定义三个 custom target（`build_mooncake_store_rust` / `_example` / `_tests`），`DEPENDS mooncake_store` 保证 C++ 静态库先就绪，再以 env 注入 `MOONCAKE_STORE_LIB_DIR` / `MOONCAKE_STORE_INCLUDE_DIR` 调 `cargo build --release`。Rust 另需 libclang（bindgen 依赖，见 `rust/README.md`）。

### 3. Go 绑定 (`mooncakestore`)

包 `mooncakestore`（`go/mooncakestore/store.go:18`）通过 CGo `#include "store_c.h"` 直接绑定。`Store` 持 `C.mooncake_store_t`，方法覆盖全量 C API：

| 类别 | 方法 |
|------|------|
| 生命周期 | `New` · `Setup` · `InitAll` · `HealthCheck` · `Close` |
| 写入 | `Put` · `PutFrom`（零拷贝）· `BatchPutFrom` |
| 读取 | `Get` · `GetInto` · `BatchGetInto` |
| 元数据 | `Exists` · `BatchExists` · `GetSize` · `Hostname` |
| 删除 | `Remove` · `RemoveByRegex` · `RemoveAll` |
| 缓冲 | `RegisterBuffer` · `UnregisterBuffer` |

`ReplicateConfig`（`config.go:18`）字段 `ReplicaNum` / `WithSoftPin` / `WithHardPin` / `PreferredSegments`，`toCConfig`（`store.go:126`）转成 `C.mooncake_replicate_config_t` 并用 `defer C.free` 管理 `CString`。错误以 `errors.go` 的哨兵值表达（`ErrStoreNil` / `ErrSetupFailed` …）。Go 绑定是两者中 API 覆盖最全的，含完整零拷贝与 batch 接口。

### 4. Rust 绑定 (`mooncake_store` crate)

crate `mooncake_store`（`Cargo.toml`，`crate-type = ["rlib"]`，依赖 `thiserror` + `libc`，build-dep `bindgen`）。`build.rs` 从 `store_c.h` 生成 `bindings.rs`，`store.rs` 内 `mod ffi` include 之（`store.rs:23`）。`MooncakeStore`（`store.rs:109`）持 `ffi::mooncake_store_t`，标 `unsafe impl Send + Sync`（底层 C 对象内部已同步）。公开方法：

| 类别 | 方法 |
|------|------|
| 生命周期 | `new` · `setup` · `health_check` |
| 读写 | `put` · `get`（返回 `Vec<u8>`）· `is_exist` · `get_size` · `get_hostname` |
| 删除 | `remove` · `remove_by_regex` · `remove_all` |
| 批量 | `batch_is_exist` |

`ReplicateConfig`（`store.rs:40`）字段 `replica_num` / `with_soft_pin` / `with_hard_pin` / `preferred_segments`，`to_ffi` 连同 owning `CString` 一起返回以防悬空。`StoreError`（`error.rs:19`）含 `NullHandle` / `InvalidString` / `OperationFailed(i32)` / `NotFound` / `InvalidArgument`，用 `thiserror` 派生。相对 Go，Rust 公共面更精简（不直接暴露 `put_from`/`register_buffer` 等），偏重 safe 抽象；批量/零拷贝路径在内部使用或留待后续扩展。

### 5. 用途

Go 绑定便于云原生/运维侧（Go 生态的调度器、sidecar、Conductor 周边）直接读写 Store，无需经 Python。Rust 绑定面向对内存安全与低开销有同时要求的 Rust 应用（如自研推理调度器、性能敏感的数据管线）。二者均要求运行时 `LD_LIBRARY_PATH` 能找到 CMake 产出的各 `.so`（`rust/README.md:53` 列出 common/etcd/store/cachelib/te/base 等路径）。

### 6. 测试

- Go：`go/tests/integration_test.go` 需运行中的 metadata server 与 `mooncake_master`。
- Rust：`rust/tests/minimal_smoke.rs` 由 `MC_RUST_STORE_RUN_INTEGRATION=true` 等环境变量驱动（`README.md:71`）；benchmark 经 `MC_RUST_BENCH_*` 缩小规模快速冒烟（`README.md:87`）。

## 四、交叉引用

- Python 绑定（`store_py.cpp` / `engram_store_py.cpp` 等 C++ 扩展）位于集成层，见 [`../08-integration/index.md`](../08-integration/index.md)，最终经 [`../09-wheel/index.md`](../09-wheel/index.md) 打包为 pip 轮子。
- 绑定包装的对象级复制策略（`ReplicateConfig` 字段语义、soft/hard pin）见 [replication-scheduling.md](replication-scheduling.md)。
- 构建/依赖约束（`WITH_STORE_GO`/`WITH_STORE_RUST` 与元数据/HA 互斥项）见 [`../00-overview/dependency-graph.md`](../00-overview/dependency-graph.md) 的构建开关矩阵。
- Rust 构建与集成冒烟说明：`mooncake-store/rust/README.md`。

## 五、关键源码入口

| 入口 | 定位 |
|------|------|
| 共用 C API 契约 | `mooncake-store/include/store_c.h:25` |
| `mooncake_replicate_config_t` | `mooncake-store/include/store_c.h:27` |
| C API 实现 | `mooncake-store/src/store_c.cpp` |
| `WITH_STORE_GO` / `WITH_STORE_RUST` 开关 | `CMakeLists.txt:17` · `CMakeLists.txt:20` |
| Go CMake 集成 `build_store_go` | `CMakeLists.txt:181` |
| Go CGo 构建脚本 | `mooncake-store/go/build.sh:33` |
| Go `Store` 类型与方法 | `mooncake-store/go/mooncakestore/store.go:30` |
| Go `ReplicateConfig` | `mooncake-store/go/mooncakestore/config.go:18` |
| Rust CMake 三个 target | `mooncake-store/rust/CMakeLists.txt:7` |
| Rust bindgen `build.rs` | `mooncake-store/rust/build.rs` |
| Rust `MooncakeStore` safe 包装 | `mooncake-store/rust/src/store.rs:109` |
| Rust `ReplicateConfig` | `mooncake-store/rust/src/store.rs:40` |
| Rust `StoreError` 枚举 | `mooncake-store/rust/src/error.rs:19` |
| Rust 集成冒烟测试 | `mooncake-store/rust/tests/minimal_smoke.rs` |
| Rust 构建说明 | `mooncake-store/rust/README.md` |
