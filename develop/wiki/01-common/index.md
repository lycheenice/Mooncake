# mooncake-common 基础公共层

## 一句话定位

`mooncake-common` 是被所有子系统依赖的**基础公共层**：它既不实现传输也不实现存储语义，而是提供三类横切能力——(1) 公共头文件（配置加载 / 环境变量 / 时间工具），(2) cmake 查找模块与构建辅助脚本，(3) `etcd` 与 `k8s-lease` 两个以 Go c-shared 形式打包的可复用后端。在树形拓扑中它是**最底层**，Transfer Engine / Store / EP / PG / integration / wheel 全部直接或间接 `include_directories(mooncake-common/include)`。

## 目录 / 源码结构

```
mooncake-common/
├── include/                 公共头（C++ 内联/模板与类声明）
│   ├── default_config.h     DefaultConfig：YAML/JSON 统一配置读取
│   ├── environ.h            Environ：MC_* 环境变量单例
│   └── duration_utils.h     ParseDurationMs：时长字符串解析
├── src/                     C++ 实现与 asio 单独编译
│   ├── default_config.cpp
│   ├── environ.cpp
│   ├── asio_impl.cpp        asio separate compilation → libasio.so
│   └── CMakeLists.txt       构建 mooncake_common / asio_shared 目标
├── tests/                   gtest 单测（default_config_test · environ_test）
├── common.cmake             全局编译标准 / option 矩阵 / 硬件 SDK 探测
├── limit_jobs.cmake         内存感知的编译/链接并行度
├── FindJsonCpp.cmake        jsoncpp 查找（CONFIG → PkgConfig → 手动）
├── FindGLOG.cmake           glog 查找
├── FindMpi.cmake            mpich 查找
├── FindUrma.cmake           FetchContent 拉取 UMDK 头（UB 协议）
├── SetupPython.cmake        解析 Python 解释器
├── SetupPyTorchEnv.cmake    EP/PG 扩展构建用的 PyTorch wheel 安装函数
├── etcd/                    Go c-shared etcd 客户端封装 → libetcd_wrapper.so
│   ├── etcd_wrapper.go      TE / Store / Snapshot 三套 client + lease/watch
│   ├── CMakeLists.txt       add_custom_command 调 go build -buildmode=c-shared
│   ├── build.sh             手动构建脚本
│   ├── go.mod / go.sum      go.etcd.io/etcd/client/v3 v3.5.21
├── k8s-lease/               Go c-shared K8s Lease leader election → libk8s_lease_wrapper.so
│   ├── k8s_lease_wrapper.go leaderelection + Lease watch + Pod label patch
│   ├── cmd/envtest-server/  本地 envtest 测试用 K8s 控制面
│   ├── k8s_lease_*_test.go  Go 端集成测试
│   ├── CMakeLists.txt       add_custom_command 调 go build -buildmode=c-shared
│   ├── go.mod / go.sum      k8s.io/client-go v0.34.3
└── CMakeLists.txt           子目录派发：etcd / k8s-lease / src / tests
```

顶层接入见 `CMakeLists.txt:7-9`（include 三个 cmake 模块）、`CMakeLists.txt:24`（`SetupPython.cmake`）、`CMakeLists.txt:32-44`（`USE_ETCD` / `STORE_USE_ETCD` 编译宏）、`CMakeLists.txt:49-58`（`STORE_USE_K8S_LEASE` 及与 etcd 的互斥校验）、`CMakeLists.txt:71-74`（`add_subdirectory(mooncake-common)` 与三个 `include_directories`）。

## 核心概念

### (a) 公共头文件：配置 / 环境 / 时间工具

**`DefaultConfig`**（`mooncake-common/include/default_config.h:16`）是 Mooncake 运行时配置的统一入口。它通过扩展名自动分发到 YAML（`loadFromYAML`）或 JSON（`loadFromJSON`）后端，将嵌套 map/array 扁平化为 `std::unordered_map<std::string, Node>`，键用点号拼接（`a.b.c`）。对外提供类型化的 `GetInt32` / `GetUInt64` / `GetDouble` / `GetBool` / `GetString` 系列 getter，每个都带 `default_value` 兜底。`GetDurationMs`（`mooncake-common/src/default_config.cpp:167`）额外复用 `ParseDurationMs`，支持 `"30s"` / `"5m"` / `"1h"` / `"500ms"` 等带单位字符串与裸毫秒数。头文件还内联定义 `init_ylt_log_level()`（`mooncake-common/include/default_config.h:152`），根据环境变量 `MC_YLT_LOG_LEVEL` 设置 `ylt/easylog` 的最小日志级别。

**`Environ`**（`mooncake-common/include/environ.h:9`）是线程安全的单例（`Environ::Get()`），在构造期（`mooncake-common/src/environ.cpp:89`）一次性读取全部 `MC_*` 环境变量并缓存为成员，避免热路径反复 `getenv`。覆盖范围包括：RDMA/IB 参数（`MC_IB_PORT`/`MC_GID_INDEX`/`MC_SLICE_SIZE`/`MC_NUM_QP_PER_EP` 等）、端点与握手（`MC_MIN_RPC_PORT`/`MC_MAX_RPC_PORT`/`MC_HANDSHAKE_LISTEN_BACKLOG`）、传输策略开关（`MC_FORCE_TCP`/`MC_FORCE_HCA`/`MC_INTRA_NVLINK`/`MC_PATH_ROUNDROBIN`）、EFA（`MC_EFA_CQ_THREADS`）、Redis（`MC_REDIS_PASSWORD`/`MC_REDIS_DB_INDEX`）以及一组 `MOONCAKE_AWS_*` S3 客户端配置（供 `s3_helper` 消费）。同时暴露静态工具 `GetInt`/`GetInt64`/`GetSizeT`/`GetBool`/`GetString` 供其它代码按需读取自定义环境变量，均带非法值告警与默认值回退。

**`duration_utils.h`**（`mooncake-common/include/duration_utils.h:23`）提供 `ParseDurationMs` 与 `TrimAsciiWhitespace`，独立的 header-only 工具，被 `DefaultConfig::GetDurationMs` 复用，也被 Store 配置侧直接调用。支持 `ms`/`s`/`m`/`h` 后缀与无后缀（裸毫秒），对溢出做显式检测。

这三个头文件编译进 `mooncake_common` 静态/共享库（`mooncake-common/src/CMakeLists.txt:53`），链 `yaml-cpp` + `jsoncpp`。同目录的 `asio_impl.cpp`（`mooncake-common/src/asio_impl.cpp:1`）通过 `ASIO_SEPARATE_COMPILATION` + `ASIO_DYN_LINK` 把 asio 源码编进独立的 `asio_shared` 共享库（`mooncake-common/src/CMakeLists.txt:30`），避免多个 shared lib 各自静态编译 asio 造成 ODR 违规。

### (b) cmake 查找模块与构建辅助

`common.cmake`（`mooncake-common/common.cmake:1`）是**全局构建策略中枢**，在顶层 `CMakeLists.txt:9` 被 include，作用于全仓。它定义：

- 编译标准：`C` 99 / `CXX` 20 / `CUDA` 20，`-Wall -Wextra -fPIC`，GCC 启用 `-fcoroutines`，Clang 启用 `-Werror=thread-safety`，Release `-O3`、Debug `-O0`，默认 `RelWithDebInfo`。
- **option 矩阵**：硬件 SDK（`USE_CUDA`/`USE_HIP`/`USE_MLU`/`USE_MACA`/`USE_MUSA`/`USE_HYGON`/`USE_COREX`/`USE_SUNRISE`/`USE_ASCEND`/`USE_ASCEND_DIRECT`/`USE_UBSHMEM`）、传输协议（`USE_TCP`/`USE_BAREX`/`USE_CXL`/`USE_EFA`/`USE_MNNVL`/`USE_NVMEOF`/`USE_UB`/`USE_CXI`）、元数据后端（`USE_ETCD`/`USE_ETCD_LEGACY`/`USE_REDIS`/`USE_HTTP`）、存储后端（`USE_3FS`）、TENT（`USE_TENT`/`USE_TPU`）、metrics（`WITH_METRICS`）、LRU master（`USE_LRU_MASTER`）等，并负责为每个开启的选项探测 SDK 路径、`include_directories`/`link_directories`、`add_compile_definitions`。
- 工具链：`ENABLE_DEBUG_SYMBOLS`（默认 ON）、`ENABLE_ASAN`、`ENABLE_SCCACHE`（`common.cmake:56`）、eRDMA 常量 `CONFIG_ERDMA`（`common.cmake:49`）、`GLOG_USE_GLOG_EXPORT`、`YLT_ENABLE_IBV`、`CMAKE_EXPORT_COMPILE_COMMANDS`，以及 `find_package(yaml-cpp/gflags/yalantinglibs)`。
- 在 `common.cmake:54` include `limit_jobs.cmake`。

`limit_jobs.cmake`（`mooncake-common/limit_jobs.cmake:1`）做**内存感知的构建并行度**：用 `cmake_host_system_information` 读取可用物理内存与逻辑核数，按每编译作业 `MAX_COMPILER_MEMORY_MB`（默认 1500 MB）、每链接作业 `MAX_LINKER_MEMORY_MB`（默认 4000 MB）估算安全并行度，clamp 到 `[1, nproc]`，Ninja 下建立 `compile_pool`/`link_pool` job pool，使高内存的链接被限流而编译仍可占满核。用户可用 `-DPARALLEL_COMPILE_JOBS=` / `-DPARALLEL_LINK_JOBS=` 覆盖。

查找模块遵循统一模式——优先 `find_package(... CONFIG QUIET)`，回退 `pkg-config`，再回退 `find_path`+`find_library`，最终合成一个 `IMPORTED INTERFACE` 目标供全仓 `target_link_libraries` 使用：

| 模块 | 文件 | 目标 | 用途 |
|------|------|------|------|
| `FindJsonCpp.cmake` | `mooncake-common/FindJsonCpp.cmake:1` | `JsonCpp::JsonCpp` | jsoncpp（配置/序列化） |
| `FindGLOG.cmake` | `mooncake-common/FindGLOG.cmake:1` | `glog::glog` | glog 日志 |
| `FindMpi.cmake` | `mooncake-common/FindMpi.cmake:1` | `MPI::MPI` | mpich（被 TENT/PG 选用） |
| `FindUrma.cmake` | `mooncake-common/FindUrma.cmake:1` | — | `FetchContent` 拉取 openEuler UMDK 头，供 UB 协议传输 |

构建辅助脚本还包含 `SetupPython.cmake`（`mooncake-common/SetupPython.cmake:1`，解析 `Python3_EXECUTABLE` 并写入 legacy `PYTHON_EXECUTABLE`，供顶层 `CMakeLists.txt:24` 与 pybind11 使用）与 `SetupPyTorchEnv.cmake`（`mooncake-common/SetupPyTorchEnv.cmake:1`，提供 `install_pytorch_wheel(version, cuda_major, cuda_minor, prefix)` 函数，按 CUDA 版本选择 `cu126`/`cu128`/`cu130` wheel，供 `BuildEpExt.cmake`/`BuildPgExt.cmake` 构建 EP/PG 扩展时安装匹配的 PyTorch）。

### (c) etcd 与 k8s-lease 两个可复用 Go c-shared 后端

两个后端都用 **Go 编译为 C 共享库**（`-buildmode=c-shared`），产出 `.so` + 自动生成的 `.h`，C++ 侧通过 `extern "C"` 直接调用，避免在 C++ 里重写一套 etcd v3 / K8s client-go。

**etcd 封装**（`mooncake-common/etcd/etcd_wrapper.go`，依赖 `go.etcd.io/etcd/client/v3 v3.5.21`）维护**三套独立 client**，各自配置、互不影响（`etcd_wrapper.go:61-81`）：

- `globalClient` —— Transfer Engine 元数据用，引用计数（`NewEtcdClient` at `etcd_wrapper.go:122`，`EtcdCloseWrapper` at `etcd_wrapper.go:224`），提供 `EtcdPutWrapper`/`EtcdGetWrapper`/`EtcdDeleteWrapper` 简单 KV。
- `storeClient` —— Store 元数据与 HA 用，带 keepalive/lease/watch/Txn（`NewStoreEtcdClient` at `etcd_wrapper.go:243`），提供 `EtcdStoreCreateWrapper`（CAS on `CreateRevision==0`，`etcd_wrapper.go:724`）、`EtcdStoreBatchCreateWrapper`（事务批量创建，`etcd_wrapper.go:751`）、`EtcdStoreGetRangeAsJsonWrapper`（范围查询序列化为 JSON，`etcd_wrapper.go:841`）、`EtcdStoreGrantLeaseWrapper`/`EtcdStoreKeepAliveWrapper`（lease + 主动续约，`etcd_wrapper.go:371`/`etcd_wrapper.go:624`）、`EtcdStoreWatchUntilDeletedWrapper`（阻塞等删除，`etcd_wrapper.go:536`）以及 `EtcdStoreWatchWithPrefixFromRevisionWrapper`（带回调的前缀 watch，`etcd_wrapper.go:967`，通过 cgo trampoline `call_watch_cb` 把 PUT/DELETE 事件回传 C++，含 `WATCH_BROKEN` 通知与 `WithCreatedNotify` 握手确保 watch 服务端已建立）。
- `snapshotClient` —— HA 快照专用（`NewSnapshotEtcdClient` at `etcd_wrapper.go:299`），`MaxCallSendMsgSize`/`MaxCallRecvMsgSize` 提升到 2 GB（`snapshotMaxMsgSize` at `etcd_wrapper.go:85`），超时 60 s，供 Store HA 的 GB 级 snapshot 读写（`SnapshotStorePutWrapper`/`SnapshotStoreGetWrapper`/`SnapshotStoreDeleteWrapper`）。

构建入口 `mooncake-common/etcd/CMakeLists.txt:1` 用 `add_custom_command` 调 `go mod tidy && go build -buildmode=c-shared`，产物 `libetcd_wrapper.so` + 拷贝 `libetcd_wrapper.h` 回源码目录，封装为 `build_etcd_wrapper` 自定义目标。

**k8s-lease 封装**（`mooncake-common/k8s-lease/k8s_lease_wrapper.go`，依赖 `k8s.io/client-go v0.34.3`）基于 K8s `coordination/v1.Lease` 与 `client-go/tools/leaderelection` 实现 **leader election**。客户端初始化（`initClient` at `k8s_lease_wrapper.go:83`）先试 `rest.InClusterConfig()`，失败回退 `KUBECONFIG` 再回退 `~/.kube/config`。导出 API：

- `K8sLeaseInit`（`k8s_lease_wrapper.go:222`）懒初始化 clientset。
- `K8sLeaseRunElection`（`k8s_lease_wrapper.go:231`）→ `runElection`（`k8s_lease_wrapper.go:115`）用 `LeaseLock` 启动 `LeaderElector` goroutine，`OnStartedLeading` 关闭 `elected` 通道、`OnStoppedLeading` 关闭 `lost` 并自清理。
- `K8sLeaseWaitElected`/`K8sLeaseWaitLost`/`K8sLeaseCancelElection`（`k8s_lease_wrapper.go:250`/`:286`/`:311`）同步等待选举结果或主动让位。
- `K8sLeaseGetHolder`（`k8s_lease_wrapper.go:331`）读取当前 holder 与 `LeaseTransitions`，并把**已过期的 lease 视作无 holder**（`k8s_lease_wrapper.go:213`），触发 standby 重新竞选。
- `K8sLeaseWatchHolder`（`k8s_lease_wrapper.go:362`）watch Lease 变更，经 cgo trampoline `call_holder_change_cb` 把 holder/transitions 回调给 C++。
- `K8sPatchPodLabel`/`K8sRemovePodLabel`（`k8s_lease_wrapper.go:521`/`:535`）用 JSON merge patch 打/删 Pod label，供 HAController 通过 label 做流量路由。

构建入口 `mooncake-common/k8s-lease/CMakeLists.txt:1` 同样以 `go build -buildmode=c-shared` 产出 `libk8s_lease_wrapper.so` + 头，封装为 `build_k8s_lease_wrapper` 目标。`cmd/envtest-server/` 提供本地 K8s envtest 控制面用于 Go 端集成测试（`k8s_lease_integration_test.go`）。

**互斥约束**：两个后端都生成 Go c-shared 库并在同一进程加载，因此顶层 `CMakeLists.txt:49-58` 强制 `STORE_USE_K8S_LEASE` 与 `STORE_USE_ETCD`、与 non-legacy `USE_ETCD` **不能同时开启**，否则 `FATAL_ERROR`。`mooncake-common/CMakeLists.txt:1-7` 据此条件 `add_subdirectory`：`(USE_ETCD AND NOT USE_ETCD_LEGACY) OR STORE_USE_ETCD` 选 etcd，`STORE_USE_K8S_LEASE` 选 k8s-lease。

## 交叉引用

- 元数据 / HA 横向复用全景见 [`../00-overview/dependency-graph.md`](../00-overview/dependency-graph.md) 的「元数据 / HA 横向」一节，其中 `STORE_USE_K8S_LEASE` 与 `STORE_USE_ETCD` 的互斥约束在构建开关矩阵中列出。
- 本层在子系统树中的位置见 [`../00-overview/architecture.md`](../00-overview/architecture.md) 的「mooncake-common（基础公共层）」段与 Tensor-Centric 全栈定位图最底层。
- etcd / k8s-lease 在 Store 端的 C++ 薄封装为 `mooncake-store/include/etcd_helper.h:16`（`EtcdHelper`）与 `mooncake-store/include/k8s_lease_helper.h:11`（`K8sLeaseHelper`），再上层接入 Store HA 的 `ha/leadership/backends/etcd/etcd_leader_coordinator.cpp` 与 `ha/leadership/backends/k8s/k8s_leader_coordinator.cpp`；该链路的完整分析见 [`../03-store/metadata-ha.md`](../03-store/metadata-ha.md)。
- `environ.h` 中 `MC_*` 环境变量对传输栈行为的影响，详见 [`../02-transfer-engine/`](../02-transfer-engine/)（RDMA/TCP/EFA 参数）；对 Store 行为的影响详见 [`../03-store/`](../03-store/)。
- 用户向的部署 / 配置说明（包括 `mooncake.json`/`mooncake.yaml` 字段、`MC_*` 环境变量表）由官方文档维护，见 `../../docs/source/install/configuration.rst` 与 `../../docs/source/design/architecture.md`，本页不重复。

## 关键源码入口

公共头与实现：

- `mooncake-common/include/default_config.h:16` — `DefaultConfig` 类（YAML/JSON 统一配置）。
- `mooncake-common/include/default_config.h:152` — `init_ylt_log_level()`（`MC_YLT_LOG_LEVEL` → easylog 级别）。
- `mooncake-common/src/default_config.cpp:22` — `DefaultConfig::Load()`（按扩展名分发 YAML/JSON）。
- `mooncake-common/src/default_config.cpp:167` — `DefaultConfig::GetDurationMs()`（复用 `ParseDurationMs`）。
- `mooncake-common/include/environ.h:9` — `Environ` 单例类声明。
- `mooncake-common/src/environ.cpp:89` — `Environ::Environ()`（读取全部 `MC_*` / `MOONCAKE_AWS_*` 环境变量）。
- `mooncake-common/include/duration_utils.h:23` — `ParseDurationMs()`（header-only 时长解析）。
- `mooncake-common/src/asio_impl.cpp:1` — asio separate compilation（避免 ODR 违规）。
- `mooncake-common/src/CMakeLists.txt:53` — `mooncake_common` 目标（链 `yaml-cpp` + `jsoncpp`）。
- `mooncake-common/src/CMakeLists.txt:30` — `asio_shared` 共享库目标。

cmake 模块与构建辅助：

- `mooncake-common/common.cmake:1` — 全局编译标准 / option 矩阵 / 硬件 SDK 探测。
- `mooncake-common/common.cmake:54` — include `limit_jobs.cmake`。
- `mooncake-common/limit_jobs.cmake:1` — 内存感知编译/链接并行度（Ninja job pool）。
- `mooncake-common/FindJsonCpp.cmake:1` — `JsonCpp::JsonCpp` 查找。
- `mooncake-common/FindGLOG.cmake:1` — `glog::glog` 查找。
- `mooncake-common/FindMpi.cmake:1` — `MPI::MPI`（mpich）查找。
- `mooncake-common/FindUrma.cmake:1` — `FetchContent` 拉取 UMDK 头（UB 协议）。
- `mooncake-common/SetupPython.cmake:1` — Python 解释器解析。
- `mooncake-common/SetupPyTorchEnv.cmake:1` — `install_pytorch_wheel()` 函数（EP/PG 扩展构建）。

etcd 后端：

- `mooncake-common/etcd/CMakeLists.txt:1` — `go build -buildmode=c-shared` → `libetcd_wrapper.so`。
- `mooncake-common/etcd/etcd_wrapper.go:122` — `NewEtcdClient`（TE client，引用计数）。
- `mooncake-common/etcd/etcd_wrapper.go:243` — `NewStoreEtcdClient`（Store client）。
- `mooncake-common/etcd/etcd_wrapper.go:299` — `NewSnapshotEtcdClient`（HA 快照 client，2 GB msg size）。
- `mooncake-common/etcd/etcd_wrapper.go:724` — `EtcdStoreCreateWrapper`（CAS 创建）。
- `mooncake-common/etcd/etcd_wrapper.go:967` — `EtcdStoreWatchWithPrefixFromRevisionWrapper`（前缀 watch + cgo 回调）。

k8s-lease 后端：

- `mooncake-common/k8s-lease/CMakeLists.txt:1` — `go build -buildmode=c-shared` → `libk8s_lease_wrapper.so`。
- `mooncake-common/k8s-lease/k8s_lease_wrapper.go:83` — `initClient`（in-cluster → KUBECONFIG → ~/.kube/config）。
- `mooncake-common/k8s-lease/k8s_lease_wrapper.go:115` — `runElection`（`LeaseLock` + `LeaderElector` goroutine）。
- `mooncake-common/k8s-lease/k8s_lease_wrapper.go:222` — `K8sLeaseInit`。
- `mooncake-common/k8s-lease/k8s_lease_wrapper.go:362` — `K8sLeaseWatchHolder`（Lease watch + cgo 回调）。
- `mooncake-common/k8s-lease/k8s_lease_wrapper.go:521` — `K8sPatchPodLabel`（Pod label 流量路由）。

顶层接入与互斥：

- `CMakeLists.txt:7-9` — include `FindJsonCpp` / `FindGLOG` / `common.cmake`。
- `CMakeLists.txt:24` — include `SetupPython.cmake`。
- `CMakeLists.txt:32-44` — `USE_ETCD` / `STORE_USE_ETCD` 编译宏。
- `CMakeLists.txt:49-58` — `STORE_USE_K8S_LEASE` 定义与 etcd 互斥校验。
- `CMakeLists.txt:71-74` — `add_subdirectory(mooncake-common)` + etcd/k8s-lease/include 的 `include_directories`。
- `mooncake-common/CMakeLists.txt:1-7` — etcd / k8s-lease 条件 `add_subdirectory`。

Store 侧消费入口（C++ 薄封装）：

- `mooncake-store/include/etcd_helper.h:16` — `EtcdHelper`（封装 `libetcd_wrapper`）。
- `mooncake-store/include/k8s_lease_helper.h:11` — `K8sLeaseHelper`（封装 `libk8s_lease_wrapper`）。
- `mooncake-store/src/ha/leadership/backends/etcd/etcd_leader_coordinator.cpp` — etcd leader 后端。
- `mooncake-store/src/ha/leadership/backends/k8s/k8s_leader_coordinator.cpp` — k8s leader 后端。
