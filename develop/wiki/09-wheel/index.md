# 打包分发层（mooncake-wheel）

## 定位

`mooncake-wheel` 是整栈的 **pip 打包出口**。发行名 `mooncake-transfer-engine`，一个 Python 包内聚合了 Transfer Engine、Store、EP、PG 的预编译扩展模块（由 [`../08-integration/index.md`](../08-integration/index.md) 产出）以及一组纯 Python 模块（CLI、Store REST 服务、vLLM connector、EP/PG 绑定、SSD 工具、配置与诊断工具）。它在架构图中处于最顶"分发层"，直接面向外部生态（SGLang / vLLM / TRT-LLM …）。

wheel 按 CUDA / 硬件平台分成多个 PyPI 变体，README 徽章（`README.md:23-26`）列出四个主力包：

| 变体包名 | 适用平台 | 徽章 |
|----------|----------|------|
| `mooncake-transfer-engine` | CUDA <= 12.9 | `README.md:23` |
| `mooncake-transfer-engine-cuda13` | CUDA 13.0 / 13.1 | `README.md:24` |
| `mooncake-transfer-engine-non-cuda` | 无 CUDA（CPU/DRAM） | `README.md:25` |
| `mooncake-transfer-engine-npu` | 华为 Ascend NPU | `README.md:26` |

安装命令（`README.md:205-212`）：

```bash
pip install mooncake-transfer-engine          # CUDA < 13
pip install mooncake-transfer-engine-cuda13   # CUDA >= 13
```

non-CUDA / NPU 变体见 [`../../docs/source/getting_started/quick-start.md`](../../docs/source/getting_started/quick-start.md)。变体的实际重命名与 keywords/classifiers 注入由 `scripts/build_wheel.sh:176`（`BUILD_VARIANTS`）用 `sed` 改写 `pyproject.toml` 完成（non-cuda `:203` / cuda13 `:213` / npu `:222`）。

## 目录 / 源码结构

```
mooncake-wheel/
├── setup.py                  平台守卫 + manylinux tag 探测 + BinaryDistribution
├── pyproject.toml            包元数据 + [project.scripts] entry points
├── mooncake/                 纯 Python 包（被 install 到 site-packages/mooncake/）
│   ├── __init__.py           顶层导出 BufferPool / RegisteredBufferPool
│   ├── README.md             mooncake_connector_v1 用法（vllm 0.10.1~0.12.0）
│   ├── cli.py / cli_client.py / cli_bench.py   C++ 二进制透传 CLI
│   ├── mooncake_config.py                     MooncakeConfig 配置对象
│   ├── mooncake_store_service.py              aiohttp Store REST 服务
│   ├── structured_object_store.py             结构化对象 bundle 高层封装
│   ├── http_metadata_server.py                HTTP 元数据服务（etcd 替代）
│   ├── mooncake_connector_v1.py               vLLM v1 KV connector
│   ├── vllm_v1_proxy_server.py                vLLM prefill/decode 反向代理
│   ├── ep.py / pg.py                          torch 版本化 EP/PG 分派
│   ├── mooncake_ep_buffer.py                  EP Buffer / EventOverlap
│   ├── mooncake_elastic_buffer.py             DeepEP elastic 兼容 ElasticBuffer
│   ├── mooncake_ssd_register.py               SSD 注册（USE_NOF）
│   ├── mooncake_ssd_unregister.py             SSD 注销（USE_NOF）
│   ├── spdk_tgt_create.py                     SPDK target 远程创建（USE_NOF）
│   ├── transfer_engine_topology_dump.py       设备拓扑导出
│   └── buffer_pool.py                         BufferPool 别名（转引 store 扩展）
└── tests/                    pytest + unittest 测试集（含 conftest.py 路径注入）
```

## 核心概念

### 1. 包内模块职责分组

| 分组 | 模块 | 职责 |
|------|------|------|
| CLI 入口 | `cli.py:12` / `cli_client.py:12` / `cli_bench.py:12` | 透传到包内 `mooncake_master` / `mooncake_client` / `transfer_engine_bench` 二进制 |
| 配置 | `mooncake_config.py:170`（`MooncakeConfig`） | 协议 / 设备 / 段大小等环境变量与配置文件统一对象 |
| 存储服务 | `mooncake_store_service.py:817`（`sync_main`） | aiohttp REST server，对外暴露 put/get/remove，entry `mc_store_rest_server` |
| 结构化对象 | `structured_object_store.py:30`（`BundleStore`） | metadata + named buffers 编码为 bundle，支持 auto/batch/parallel put 与 zero_copy |
| 元数据服务 | `http_metadata_server.py:31`（`KVBootstrapServer`） | 简易 HTTP KV，作为 etcd 替代，entry `mooncake_http_metadata_server` |
| vLLM 连接器 | `mooncake_connector_v1.py:122`（`MooncakeConnector`） | vLLM `KVConnectorBase_V1` 实现，producer/consumer 双角色 |
| vLLM 代理 | `vllm_v1_proxy_server.py:87`（`app`） | FastAPI 反向代理，轮询 prefill/decode 实例 |
| EP / PG 绑定 | `ep.py:10` / `pg.py:10` | 按 torch 版本动态 `import mooncake.ep_<ver>` / `pg_<ver>` |
| EP Buffer | `mooncake_ep_buffer.py:82`（`Buffer`） / `:24`（`EventOverlap`） | EP dispatch/combine 的通信缓冲与 CUDA event 重叠 |
| Elastic Buffer | `mooncake_elastic_buffer.py:84`（`ElasticBuffer`） / `:35`（`EPHandle`） | 官方 DeepEP elastic API 兼容层 |
| SSD 工具 | `mooncake_ssd_register.py:18` / `mooncake_ssd_unregister.py:17` / `spdk_tgt_create.py:14` | SPDK/NVMe-oF target 注册/注销/创建（仅 `USE_NOF` 构建） |
| 诊断 | `transfer_engine_topology_dump.py:19` | 导出本机设备拓扑，entry `transfer_engine_topology_dump` |
| buffer_pool | `buffer_pool.py:6` | `BufferPool`/`RegisteredBufferPool` 别名，转引 integration 的 `store.BufferPool` |

### 2. entry points

`pyproject.toml:46` 的 `[project.scripts]` 暴露 6 个命令，均指向 `mooncake` 包内的 `main` / `sync_main`：

```
mooncake_master              → mooncake.cli:main
mooncake_client              → mooncake.cli_client:main
transfer_engine_bench        → mooncake.cli_bench:main
mooncake_http_metadata_server→ mooncake.http_metadata_server:main
mc_store_rest_server         → mooncake.mooncake_store_service:sync_main
transfer_engine_topology_dump→ mooncake.transfer_engine_topology_dump:main
```

### 3. 纯 Python 包 + 预编译 .so

`setup.py` 只打 Python；所有 `.so`（`engine.so` / `store.so` / `ep.so` / `pg.so` / `libtransfer_engine.so` / `nvlink_allocator.so` / `ubshmem_fabric_allocator.so` / `ascend_transport.so` 等）由两条路径汇入 `mooncake-wheel/mooncake/`：

1. integration 的 CMake `install` 把它们落到 `site-packages/mooncake/`（见 [`../08-integration/index.md`](../08-integration/index.md)）。
2. `scripts/build_wheel.sh:78-120` 在打包前 `cp` 这些 `.so` 与 `transfer_engine_bench` 二进制进 `mooncake-wheel/mooncake/`，再由 `setup.py` 连同 `.py` 一起打成 wheel（`package-data = ["*.so", ...]`，`pyproject.toml:62`）。

### 4. binary wheel 标记

`setup.py` 用 `BinaryDistribution`（`:152`，`has_ext_modules` 返回 `True`）+ `CustomBdistWheel`（`:157`，`root_is_pure=False` 且 `plat_name=get_platform()`）强制生成 platform wheel而非 pure wheel。`_detect_manylinux_tag`（`:26`）通过 `packaging.tags.glibc_version_string` / `getconf GNU_LIBC_VERSION` / ctypes `gnu_get_libc_version` 三级回退探测 glibc 版本，生成 PEP 600 风格 `manylinux_<major>_<minor>` tag；`get_arch`（`:109`）支持 `x86_64` 与 `aarch64`。平台守卫在 `:9` 拒绝 `win32` / `darwin`。

### 5. torch 版本化的 EP/PG

`ep.py` / `pg.py` 在 import 时读 `torch.__version__`，拼出 `mooncake.ep_<ver>`（如 `mooncake.ep_2_8_0`）动态加载（`ep.py:10`）。因此**每个 wheel 只对应一个 torch 版本的 EP/PG `.so`**；`ep.py` 还会顺带加载兼容的 `pg_<ver>` 模块（`ep.py:19`），保证 `from mooncake import ep` 即同时拿到 EP 与 PG 符号。

### 6. 结构化对象存储

`structured_object_store.py`（4548 行）是纯 Python 的高层封装：把一个结构化对象（metadata + 多个 named buffer）编码为可分片的 bundle，`BundleStore` Protocol（`:30`）定义 `put/get/remove`，`BundleTransferPolicy`（`:39`）控制 `auto/batch/parallel` put 与 `auto/zero_copy/copy` 模式。它底层调用 integration 的 `MooncakeDistributedStore`，是上层框架（如 connector）偏好使用的接口。

### 7. vLLM connector 与 proxy

`mooncake_connector_v1.py` 实现 vLLM v1 的 `KVConnectorBase_V1`：`MooncakeConnector`（`:122`，worker 侧）/ `MooncakeConnectorScheduler`（`:232`，scheduler 侧）/ `MooncakeConnectorWorker`（`:395`），通过 ZMQ side channel + Mooncake Store 在 prefill 与 decode 节点间搬 KV cache。`vllm_v1_proxy_server.py` 是配套 FastAPI 反向代理，轮询 prefill/decode 实例。适用 vllm 0.10.1~0.12.0；0.13.0+ 改用 vllm 内置的 intree mooncake_connector（见 `mooncake/README.md`）。对接生态详见 [`../11-ecosystem/index.md`](../11-ecosystem/index.md)。

## 交叉引用

- Python 绑定层（提供 `.so` 与 `MooncakeDistributedStore` 等被 import 的扩展）：[`../08-integration/index.md`](../08-integration/index.md)
- 外部生态对接（vLLM connector / SGLang / TRT-LLM …）：[`../11-ecosystem/index.md`](../11-ecosystem/index.md)
- 官方 Python API 参考：[`../../docs/source/api-reference/python/index.md`](../../docs/source/api-reference/python/index.md)
- 安装与快速开始：[`../../docs/source/getting_started/quick-start.md`](../../docs/source/getting_started/quick-start.md)
- 源码构建：[`../../docs/source/getting_started/build.md`](../../docs/source/getting_started/build.md)
- 顶层安装命令与变体徽章：[`../../README.md`](../../README.md)（`:205-212` 安装，`:23-26` 徽章）
- 依赖层次（wheel 为最顶分发层）：[`../00-overview/dependency-graph.md`](../00-overview/dependency-graph.md)
- 架构拓扑与 wheel 定位：[`../00-overview/architecture.md`](../00-overview/architecture.md)

## 关键源码入口

| 关注点 | 入口 |
|--------|------|
| 包元数据 / 版本 | `mooncake-wheel/pyproject.toml:24`（`name`）/ `:25`（`version`）/ `:32`（`dependencies`） |
| entry points | `mooncake-wheel/pyproject.toml:46`（`[project.scripts]`） |
| package-data（含 `.so`） | `mooncake-wheel/pyproject.toml:62`（`[tool.setuptools.package-data]`） |
| manylinux tag 探测 | `mooncake-wheel/setup.py:26`（`_detect_manylinux_tag`） |
| 平台 tag 组装 | `mooncake-wheel/setup.py:109`（`get_arch`）/ `:127`（`get_system`）/ `:144`（`get_platform`） |
| binary wheel 标记 | `mooncake-wheel/setup.py:152`（`BinaryDistribution`）/ `:157`（`CustomBdistWheel`） |
| 顶层包导出 | `mooncake-wheel/mooncake/__init__.py:3` |
| connector README | `mooncake-wheel/mooncake/README.md:1` |
| CLI 二进制透传 | `mooncake-wheel/mooncake/cli.py:12` / `cli_client.py:12` / `cli_bench.py:12` |
| 配置对象 | `mooncake-wheel/mooncake/mooncake_config.py:170`（`MooncakeConfig`） |
| Store REST 服务 | `mooncake-wheel/mooncake/mooncake_store_service.py:817`（`sync_main`）/ `:769`（`async main`） |
| 结构化对象存储 | `mooncake-wheel/mooncake/structured_object_store.py:30`（`BundleStore`）/ `:39`（`BundleTransferPolicy`） |
| HTTP 元数据服务 | `mooncake-wheel/mooncake/http_metadata_server.py:31`（`KVBootstrapServer`） |
| vLLM connector | `mooncake-wheel/mooncake/mooncake_connector_v1.py:122`（`MooncakeConnector`）/ `:232`（`Scheduler`）/ `:395`（`Worker`） |
| vLLM proxy | `mooncake-wheel/mooncake/vllm_v1_proxy_server.py:87`（`app`） |
| EP torch 版本分派 | `mooncake-wheel/mooncake/ep.py:10`（`ep` 加载）/ `:19`（兼容 `pg` 加载） |
| PG torch 版本分派 | `mooncake-wheel/mooncake/pg.py:10` |
| EP Buffer | `mooncake-wheel/mooncake/mooncake_ep_buffer.py:82`（`Buffer`）/ `:24`（`EventOverlap`） |
| Elastic Buffer | `mooncake-wheel/mooncake/mooncake_elastic_buffer.py:84`（`ElasticBuffer`）/ `:35`（`EPHandle`） |
| SSD 注册 | `mooncake-wheel/mooncake/mooncake_ssd_register.py:18`（`MooncakeNoFRegister`） |
| SSD 注销 | `mooncake-wheel/mooncake/mooncake_ssd_unregister.py:17`（`MooncakeNoFUnregister`） |
| SPDK target 创建 | `mooncake-wheel/mooncake/spdk_tgt_create.py:14`（`SPDKTgtCreator`） |
| 拓扑导出 | `mooncake-wheel/mooncake/transfer_engine_topology_dump.py:19`（`main`） |
| buffer_pool 别名 | `mooncake-wheel/mooncake/buffer_pool.py:6` |
| 测试路径注入 | `mooncake-wheel/tests/conftest.py:7` |
| wheel 构建脚本 | `scripts/build_wheel.sh:176`（`BUILD_VARIANTS`）/ `:203`（non-cuda）/ `:213`（cuda13）/ `:222`（npu）/ `:78`（`.so` 拷入） |
| 安装命令 | `README.md:205`（CUDA<13）/ `:209`（CUDA>=13） |
