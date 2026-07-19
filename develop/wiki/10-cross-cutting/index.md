# 横向支撑工具（Cross-Cutting Tooling）

## 定位

本页覆盖 Mooncake 仓库内**非 CMake 子系统但属于架构落地必要支撑**的横向工具与工程治理资产。这些目录不参与顶层 `add_subdirectory` 构建树（见 `CMakeLists.txt:71-195`），但服务于构建、测试、可观测性、部署、CI 与协作：

- **可观测性**：`monitoring/`（Grafana + Prometheus 监控栈）
- **性能基准**：`benchmarks/`（KVCache 存储 / xPyD 推理基准）
- **测试与脚本**：`scripts/`（CI 入口、Python 集成测试、T-One E2E、Ascend 工具）
- **容器化**：`docker/`（运行时镜像）· `.devcontainer/`（VSCode 远程开发容器）
- **外部子模块**：`extern/`（pybind11、yalantinglibs）· `.gitmodules`
- **CI 工作流**：`.github/workflows/`（多硬件矩阵 CI/发布流水线）
- **工程治理**：`.pre-commit-config.yaml`、`.clang-format`、`.typos.toml`、`dependencies.sh`、`MAINTAINERS.md`、`CONTRIBUTING.md`、`CODE_OF_CONDUCT.md`

这些资产横向作用于全部子系统（`mooncake-common` → `mooncake-wheel`），其与子系统构建的耦合点仅在 `dependencies.sh`、pre-commit 钩子与 CI matrix 的 `WITH_*` 开关中体现。

## 目录 / 源码结构

```
仓库根
├── monitoring/                 监控栈（独立 docker-compose）
│   ├── docker-compose.yml      Prometheus + Grafana 服务定义
│   ├── run.sh                  等价的 docker run 启动脚本
│   ├── prometheus/prometheus.yml   scrape 配置（mooncake_master :9003）
│   ├── grafana/                provisioning + dashboards
│   └── README.md
├── benchmarks/
│   ├── storage_benchmark_v1/   KVCache 存储基准（trace replay，doc/README.md）
│   └── xypd_benchmarks/        xPyD 推理基准（proxy_demo.py + vllm-benchmarks）
├── scripts/
│   ├── code_format.sh          C/C++ clang-format 包装（pre-commit + CI 共用）
│   ├── build_wheel.sh          wheel 构建入口（CI/本地共用）
│   ├── run_tests.sh            Python wheel 集成测试主入口
│   ├── test_installation.sh    wheel 安装验证
│   ├── test_async_store.py / test_tensor_api.py / test_copy_move_api.py
│   ├── test_drain_http_api.py / test_engram_store.py / test_upsert_api.py
│   ├── bench_engram_store_27b.py
│   ├── check_hicache_hugepage_requirements.py / test_hicache_hugepage_requirements.py
│   ├── generate_cluster_topology.py
│   ├── launch_cxi_transfer_test.sh
│   ├── ascend/                 Ascend 依赖与性能脚本
│   ├── management/mc_meta_cli.py  metadata server 管理 CLI
│   └── tone_tests/             T-One E2E 测试套（README_zh.md / README_en.md）
├── docker/                     运行时镜像 Dockerfile
│   ├── mooncake.Dockerfile     CUDA + EP + Store 主镜像（两阶段）
│   ├── master.Dockerfile       mooncake_master 独立镜像
│   └── musa.Dockerfile         摩尔线程 MUSA 镜像
├── .devcontainer/              VSCode Remote-Containers（experimental）
│   ├── devcontainer.json · Dockerfile · README.md
├── extern/                     git 子模块
│   ├── pybind11/               Python C++ 扩展绑定（integration 依赖）
│   └── yalantinglibs/          RPC 序列化/HTTP 库（Store/TE 依赖）
├── .github/
│   ├── workflows/              24 个 CI/发布工作流
│   ├── CODEOWNERS · labeler.yml · pull_request_template.md · ISSUE_TEMPLATE/
└── 工程治理配置
    ├── .gitmodules             pybind11 + yalantinglibs 子模块声明
    ├── .pre-commit-config.yaml 钩子编排（project hook + ruff/clang-format/cmake-format/codespell）
    ├── .clang-format           C/C++ 风格基准
    ├── .typos.toml             拼写白名单（CANN / hsa / DeepEP ue8m0x4 等）
    ├── dependencies.sh         系统包 + 子模块 + yalantinglibs + Go + SPDK 一键安装
    ├── MAINTAINERS.md          codeowner 名录
    ├── CONTRIBUTING.md         PR 前缀 / RFC / pre-commit 工作流
    └── CODE_OF_CONDUCT.md
```

## 核心概念

### 1. 监控栈（monitoring/）

`monitoring/` 是一套独立的 Docker 化 Prometheus + Grafana 组合，**唯一抓取目标是 `mooncake_master` 的 metrics 端点**（默认 `host.docker.internal:9003`，见 `monitoring/prometheus/prometheus.yml:13`）。两条等价启动路径：

- `docker-compose up -d`（声明式，`monitoring/docker-compose.yml:4` Prometheus、`:22` Grafana）
- `bash monitoring/run.sh`（命令式 `docker run`，先建网络/卷再启容器，`monitoring/run.sh:18` 起 Prometheus、`:33` 起 Grafana）

要被监控，`mooncake_master` 必须以 `--metrics_port=9003 --enable_metric_reporting=true` 启动（见 `monitoring/README.md:50-66`）；Store 的 metrics 由 `master_metric_manager` / `ha_metric_manager` 导出（参见 Store 模块页）。Grafana 通过 `grafana/provisioning/` 预置 datasource 与 dashboard，默认账号 `admin/admin`。

监控相关排障（metrics 不出、端口冲突、master 不可达等）见官方 [`../../docs/source/troubleshooting/troubleshooting.md`](../../docs/source/troubleshooting/troubleshooting.md)。

### 2. 性能基准（benchmarks/）

两类基准并存：

- **`storage_benchmark_v1/`**：KVCache 存储侧基准。`benchmark.py` 以 FAST'25 trace 回放模拟真实 cache 访问，输出 hit rate、读写带宽、p50/p95/p99 延迟。关键选项 `--scenario {conversation,synthetic,toolagent,all}`、`--model {glm5,kimi-k2.6}`、`--max-pages`（超限时启用 modulo 映射）、`--threads`（多客户端）、`--replay-scales`（变速回放）、`--fsync-mode`（持久化成本）。完整字段见 `benchmarks/storage_benchmark_v1/doc/README.md`。
- **`xypd_benchmarks/`**：xPyD 推理侧基准，含 `proxy_demo.py` 与 `vllm-benchmarks/` 子树，用于 PD 分离端到端吞吐测量。

两者均不进 CMake 构建，靠 `python` 直接运行；存储基准的可重复性指引（清缓存、固定机器、多次试跑、`--progress-interval 0`）见其 doc/README.md "Measurement Notes"。

### 3. 测试与脚本（scripts/）

`scripts/` 是**全仓库脚本资产的根**，覆盖构建/测试/格式/部署/管理：

- **构建与安装**：`build_wheel.sh`（CI 与本地共用，受 `PYTHON_VERSION` / `OUTPUT_DIR` / `FREE_BUILD_DIR` 等环境变量控制）、`test_installation.sh`（wheel 安装后冒烟）。
- **格式**：`code_format.sh` 是 pre-commit 与 CI 的共享入口，默认对相对 `origin/main` 的改动文件跑 clang-format，`--check` 仅检查不修改，`--all` 全量（`scripts/code_format.sh:1-23` 头注释）。
- **Python wheel 集成测试**：`run_tests.sh` 串起 transfer_engine、master、replicated store、dummy client、put/get tensor、safetensor、SSD offload、CXL、CLI 等子测试，是 `test-wheel-ubuntu` 作业的核心命令（`.github/workflows/ci.yml:501`）。
- **单点 API 测试**：`test_async_store.py` / `test_tensor_api.py` / `test_copy_move_api.py` / `test_drain_http_api.py` / `test_engram_store.py` / `test_upsert_api.py`，分别对应 CI 中独立 step。
- **HiCache hugepage 探测**：`check_hicache_hugepage_requirements.py` 被 `docker/mooncake.Dockerfile` 拷为 `/usr/local/bin/mooncake-hicache-sizing`（`docker/mooncake.Dockerfile:119`），`test_hicache_hugepage_requirements.py` 在 CI 中独立 step 验证（`.github/workflows/ci.yml:78`）。
- **Ascend 工具**：`ascend/dependencies_ascend.sh`、`dependencies_ascend_installation.sh`、`dependencies_openeuler.sh`、`perf/`，支撑 `ci_ascend.yml` 作业链。
- **管理 CLI**：`management/mc_meta_cli.py` 操作 metadata server。

**T-One E2E 套**（`scripts/tone_tests/`）：跑在 [T-One](https://tone.openanolis.cn) 平台的 2×A10 双机 E2E，覆盖 `test_1p1d_erdma.sh`（PD 分离）、`test_hicache_storage_mooncake_backend.sh`（HiCache+Store）、`test_disaggregation_different_tp.sh`（异构 TP PD）。统一入口 `tone_tests/scripts/run_test.sh`，用例脚本须声明 `test_case_name` / `TEST_TYPE=single|double` 并实现 `run_test()` / `parse()`，结果写 `test_results.json`。规范见 `scripts/tone_tests/README_zh.md` / `README_en.md`。

**本地 pre-PR 验证**：仓库内置 `.claude/skills/mooncake-ci-local` skill，默认入口为 `bash scripts/run_ci_test.sh`，编排 GitHub-like paths-filter、typos、`code_format.sh --check`、ASan CMake 构建、`ctest`、wheel 构建与安装、`scripts/run_tests.sh` 与 Python API 选中测试，日志落到 `local_test/run-ci-logs/<timestamp>/`。详见 [`.claude/skills/mooncake-ci-local/SKILL.md`](../../../.claude/skills/mooncake-ci-local/SKILL.md)。

### 4. 容器化（docker/ 与 .devcontainer/）

`docker/` 提供生产/部署用镜像：

- **`mooncake.Dockerfile`**：两阶段构建。builder 阶段基于 `nvidia/cuda:<ver>-devel-ubuntu`，安装 Python、跑 `dependencies.sh -y`、CMake+Ninja 配置（`USE_HTTP=ON`、`USE_ETCD=ON`、`USE_CUDA=ON`、`WITH_EP=ON`、`STORE_USE_ETCD=ON`）、构建 nvlink allocator、`scripts/build_wheel.sh` 打轮子；runtime 阶段装 `rdma-core` / `libibverbs1` / `liburing2` 等运行时依赖，拷 wheel 安装并附带 `mooncake-hicache-sizing`。`docker/mooncake.Dockerfile:9` builder、`:82` runtime、`:120` install wheel。
- **`master.Dockerfile`**：`mooncake_master` 独立镜像。
- **`musa.Dockerfile`**：摩尔线程 MUSA 镜像。

`.devcontainer/` 是 VSCode Remote-Containers **experimental** 开发环境，`devcontainer.json` + `Dockerfile` + `README.md`，提供 `cmake -DUSE_ETCD=OFF -DUSE_HTTP=ON` 的最小可构建/可测试容器（含 http metadata server 启动指引）。

### 5. 外部子模块（extern/ 与 .gitmodules）

`extern/` 是 git 子模块集中地，被 `dependencies.sh` 在安装阶段 `git submodule update --init --recursive` 拉起（`dependencies.sh:228-247`）：

- **`extern/pybind11`**：Python C++ 扩展绑定机制，`mooncake-integration` 全部 `*_py.cpp` 依赖它（顶层 `CMakeLists.txt:25` 引用）。详见 `../08-integration/`。
- **`extern/yalantinglibs`**：alibaba 的 yalantinglibs（RPC / 序列化 / HTTP），被 Store 与 TE 用于 RPC 服务。`dependencies.sh:250-272` 单独 cmake 构建+安装。

SPDK（NVMe-oF）不在 `extern/` 默认内容中，`--with-spdk` 触发时才 `git clone` 到 `extern/spdk` 并构建（`dependencies.sh:375-435`）。

子模块声明见 `.gitmodules:1`（pybind11，stable 分支）与 `.gitmodules:5`（yalantinglibs，v0.5.7 分支）。

### 6. GitHub Actions CI（.github/workflows/）

24 个工作流，组成多硬件/多发行版的 CI 与发布矩阵。主干 `ci.yml` 的关键 job：

| Job | 触发条件 | 职责 |
|-----|---------|------|
| `spell-check` | always（PR/push/dispatch） | `crate-ci/typos` 跑 `.typos.toml` 白名单 |
| `clang-format` | always | `scripts/code_format.sh --check --base origin/main` |
| `docs-check` | paths-filter `docs/**` | Sphinx `-W` 严格构建 |
| `check-paths` | always | paths-filter 决定是否跑下游 src/tent 任务 |
| `build` | src 改动 | CUDA 12.8 + Python 3.10/3.12 + ASan + coverage + ctest + Rust/Go 绑定 + wheel |
| `build-musa` | src 改动 | 摩尔线程 MUSA 容器内构建 |
| `build-arm64` | src 改动 | arm64 SBSA + CUDA 13.0 wheel |
| `build-flags` | src 改动 | 全 `WITH_*` ON 构建测试 + Rust 绑定测试 |
| `build-docker` | src 改动 | `docker/mooncake.Dockerfile` 镜像构建 |
| `test-wheel-ubuntu` | 依赖 `build-flags` | wheel 在 ubuntu-22.04/24.04 × py3.10/3.12 安装+运行 `scripts/run_tests.sh` 与多项 API 测试 |
| `build-wheel-cu13` | src 改动 | 复用 `ci_cu13.yml` |
| `build-wheel-efa` | src 改动 | 复用 `ci_efa.yml` |
| `ascend-test` | 依赖 `build` | 复用 `ci_ascend.yml` |
| `integration-test` | 依赖 `build` | 复用 `integration-test.yml` |
| `tent-ci` | tent 改动 | `USE_TENT=ON` × `{cuda-on, cuda-off}` 双跑 |
| `ci-gate` | always（汇总） | 用 jq 校验所有 needs 结果 |

`ci-gate`（`.github/workflows/ci.yml:1080`）是 PR 合并的总闸；`check-paths`（`:901`）通过 `dorny/paths-filter` 决定 src/tent 下游是否运行，是 CI 节流的关键。`publish-*` / `release-*` 系列工作流负责各平台 wheel 与 Docker 镜像的发布。

### 7. 工程治理文件

- **`dependencies.sh`**：一键安装脚本（需 root）。检测 OS（ubuntu/debian/centos/rhel/rocky/almalinux/euleros/openeuler），装系统包、同步子模块、构建安装 yalantinglibs（`:250-272`）、装 Go 1.25.9（`:288-352`，多镜像回退 + 自动设 GOPROXY）、可选 SPDK（`:375-435`）。`docker/mooncake.Dockerfile:51` 与所有 CI build job 都 `bash dependencies.sh -y`。
- **`.pre-commit-config.yaml`**：钩子编排。`pre-commit-hooks` 基础卫生（尾空白/EOF/yaml/merge-conflict/large-file，排除 `extern/` / `build/` / `FAST25-release/`）；**local mooncake-code-format**（`./scripts/code_format.sh`，`:28-34`）始终运行；`ruff` + `ruff-format`（Python，`:36-43`）；`codespell`（`:45-50`，词典忽略 `te,mooncake,KVCache,cann,hsa`）；`clang-format` v20.1.8（`:52-57`，匹配 `.\.(c|cc|cpp|cxx|h|hpp)$`）；`cmake-format`（`:59-63`）。
- **`.clang-format`**：C/C++ 风格基准，clang-format 20 对齐（CI `clang-format` job 装 `clang-format-20`）。
- **`.typos.toml`**：拼写白名单。`CANN/ASO/fre/wqs/hsa/Optin/HPE` 等 accept；`mooncake-transfer-engine/tent/.../nlohmann/json.h` 与 `mooncake-ep/include/elastic/*`（DeepEP 派生头）排除。
- **`CONTRIBUTING.md`**：PR 前缀体系（`[Bugfix]`/`[CI/Build]`/`[Doc]`/`[Integration]`/`[P2PStore]`/`[Store]`/`[TransferEngine]`/`[Misc]`，别名 `[TE]`/`[TENT]`/`[EP]`/`[PG]`/`[Wheel]` 等）；>500 LOC（不含测试）需先开 RFC issue；pre-commit 安装与 `--no-verify` 使用规范；Google C++/Python 风格。
- **`MAINTAINERS.md`**：codeowner 名录（TE/Store/SGLang 集成/LLM 生态协作四个方向）。
- **`CODE_OF_CONDUCT.md`**：社区行为准则。
- **`.github/pull_request_template.md`** 与 `AGENTS.md`：PR 描述模板与此仓库的 agent/CI 协作约定（见仓库根 `AGENTS.md`）。

## 交叉引用

- 子系统依赖层次与构建开关矩阵：[`../00-overview/dependency-graph.md`](../00-overview/dependency-graph.md)
- 架构拓扑（横向支撑定位）：[`../00-overview/architecture.md`](../00-overview/architecture.md)（"横向支撑"小节）
- Python 绑定层（pybind11 消费方）：[`../08-integration/`](../08-integration/)
- pip 打包出口（`scripts/build_wheel.sh` 的产物）：[`../09-wheel/`](../09-wheel/)
- 监控/部署排障官方文档：[`../../docs/source/troubleshooting/troubleshooting.md`](../../docs/source/troubleshooting/troubleshooting.md)
- 部署指南：[`../../docs/source/deployment/mooncake-store-deployment-guide.md`](../../docs/source/deployment/mooncake-store-deployment-guide.md)
- 本地 CI skill：[`.claude/skills/mooncake-ci-local/SKILL.md`](../../../.claude/skills/mooncake-ci-local/SKILL.md)
- 排障 skill：[`.claude/skills/mooncake-troubleshoot/SKILL.md`](../../../.claude/skills/mooncake-troubleshoot/SKILL.md)
- 外部生态（CI 覆盖的硬件矩阵驱动生态集成）：[`../11-ecosystem/`](../11-ecosystem/)
- 附录（性能基线 / FAQ）：[`../12-appendix/`](../12-appendix/)

## 关键源码入口

| 关注点 | 入口 |
|--------|------|
| 监控栈 Prometheus 定义 | `monitoring/docker-compose.yml:4` 与 `monitoring/run.sh:18` |
| 监控栈 Grafana 定义 | `monitoring/docker-compose.yml:22` 与 `monitoring/run.sh:33` |
| Prometheus scrape 目标 | `monitoring/prometheus/prometheus.yml:13`（`host.docker.internal:9003`） |
| 存储基准主入口 | `benchmarks/storage_benchmark_v1/benchmark.py`（文档 `benchmarks/storage_benchmark_v1/doc/README.md`） |
| xPyD 推理基准 | `benchmarks/xypd_benchmarks/proxy_demo.py` |
| code_format 共用入口 | `scripts/code_format.sh:1`（头注释列用法；pre-commit 与 CI 共用） |
| wheel 构建 | `scripts/build_wheel.sh` |
| Python wheel 测试主入口 | `scripts/run_tests.sh:12`（transfer_engine target/initiator 起） |
| T-One E2E 入口 | `scripts/tone_tests/scripts/run_test.sh`（规范 `scripts/tone_tests/README_zh.md`） |
| Ascend 依赖 | `scripts/ascend/dependencies_ascend.sh` |
| Metadata 管理 CLI | `scripts/management/mc_meta_cli.py` |
| 主 CI 工作流 | `.github/workflows/ci.yml:1`（`build` 在 `:19`，`ci-gate` 在 `:1080`，`check-paths` 在 `:901`） |
| CI 格式检查 job | `.github/workflows/ci.yml:809`（`clang-format` job，调 `code_format.sh --check`） |
| CI 拼写检查 job | `.github/workflows/ci.yml:793`（`spell-check` job） |
| 主 Dockerfile | `docker/mooncake.Dockerfile:9`（builder）/ `:82`（runtime）/ `:120`（wheel install） |
| 依赖安装脚本 | `dependencies.sh:81`（root 检查）/ `:228-247`（子模块）/ `:250-272`（yalantinglibs）/ `:288-352`（Go）/ `:375-435`（SPDK） |
| 子模块声明 | `.gitmodules:1`（pybind11）/ `:5`（yalantinglibs） |
| pre-commit 钩子编排 | `.pre-commit-config.yaml:28`（local code-format）/ `:52`（clang-format）/ `:59`（cmake-format） |
| 拼写白名单 | `.typos.toml:1`（accept 列表）/ `:15`（excludes） |
| PR 前缀规范 | `CONTRIBUTING.md:19`（前缀清单） / `:39`（RFC 阈值） |
| Codeowner 名录 | `MAINTAINERS.md:5` |
| 本地 CI skill | `.claude/skills/mooncake-ci-local/SKILL.md:8`（默认入口段） |
| 排障 skill | `.claude/skills/mooncake-troubleshoot/SKILL.md:13`（诊断策略起） |
