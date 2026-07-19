# Mooncake 内部架构 Wiki

本 Wiki 是 Mooncake 的**源码与架构内部知识库**，按子系统树组织，面向希望深入理解 Mooncake 内部实现的开发者。

## 定位与分工

Mooncake 已有用户向官方文档（Sphinx 站）位于仓库的 `docs/source/`，主要面向**使用、部署与 API 参考**。本 Wiki 则面向**源码结构与内部机制**，二者互补：

| 维度 | 官方 docs/ | 本 Wiki |
|------|-----------|---------|
| 受众 | 用户 / 运维 / 集成方 | 源码贡献者 / 架构研究者 |
| 焦点 | 怎么用、怎么部署、API 签名 | 怎么实现、模块如何划分、子系统如何交叉 |
| 语言 | 英文为主 | 中文为主（术语/路径/代码/API 保留英文） |

为避免重复，凡 docs/ 已覆盖的用户向内容，本 Wiki 仅以相对链接引用（从 `develop/wiki/` 指向 `../../docs/source/...`），不复制。

## 阅读顺序

1. 先读 [`00-overview/architecture.md`](00-overview/architecture.md)，掌握树形拓扑与子系统定位。
2. 再读 [`00-overview/dependency-graph.md`](00-overview/dependency-graph.md)，理解子系统间的依赖与交叉引用。
3. 按需进入各子系统目录（`01-common` → `12-appendix`）深入。

## 目录结构

```
develop/wiki/
├── README.md                         本文件
├── 00-overview/                      总览：架构拓扑 / 依赖图 / 硬件协议矩阵 / 术语表
├── 01-common/                        基础公共层（cmake / etcd / k8s-lease）
├── 02-transfer-engine/               传输引擎核心 + transport 后端 + 分配器 + TENT
├── 03-store/                         分布式存储引擎（8 个模块页）
├── 04-p2p-store/                     P2P 存储 / checkpoint-engine 原型
├── 05-ep/                            专家并行（Expert Parallelism）
├── 06-pg/                            进程组后端（Process Group）
├── 07-rl/                            RL 场景示例
├── 08-integration/                   Python 绑定层（C++ 扩展 + allocator）
├── 09-wheel/                         pip 打包分发（Python 包 + CLI）
├── 10-cross-cutting/                 横向支撑（monitoring / benchmarks / scripts / docker）
├── 11-ecosystem/                     外部生态集成
└── 12-appendix/                      错误码 / 性能基线 / 变更时间线 / FAQ
```

## 每页统一骨架

为保证一致性，每个子系统页面遵循统一结构：

1. **定位** — 该模块是什么、解决什么问题。
2. **目录 / 源码结构** — 对应仓库目录与关键文件。
3. **核心概念** — 关键抽象、数据流、生命周期。
4. **交叉引用** — 与其它子系统 / docs/ 的关联（带链接）。
5. **关键源码入口** — 以 `path:line` 形式标注入口点，便于跳转。

> 代码引用采用 `相对路径:行号` 格式，例如 `mooncake-transfer-engine/src/transfer_engine.cpp:42`。

## 约定

- 正文中文；术语、源码路径、命令、API、配置项保留英文。
- 不新增客户端 JS；纯 Markdown，便于检索与 diff。
- 不复制 docs/ 内容，只做结构化引用。
- 保持链接相对且有效。

## 本地预览（mkdocs）

本 Wiki 配置了 [mkdocs](https://www.mkdocs.org/) + [mkdocs-material](https://squidfunk.github.io/mkdocs-material/) 主题，可本地渲染为带导航 / 搜索 / 暗色模式的站点。配置见 [`mkdocs.yml`](mkdocs.yml)，依赖见 [`requirements-docs.txt`](requirements-docs.txt)。

```bash
cd develop/wiki
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements-docs.txt
mkdocs serve -f mkdocs.yml          # 开发预览: http://127.0.0.1:8000
mkdocs build -f mkdocs.yml -d site  # 构建静态站点到 site/
```

> 注：Wiki 页面中以 `../../docs/source/...` 引用的官方文档链接在 GitHub 上可直接跳转；mkdocs 构建时这些页不在 `docs_dir` 内，会以告警形式提示但不影响渲染。这些链接的正确归宿是官方 Sphinx 站（<https://kvcache-ai.github.io/Mooncake/>）。

---

Mooncake 主仓库：https://github.com/kvcache-ai/Mooncake · FAST'25 Best Paper。
