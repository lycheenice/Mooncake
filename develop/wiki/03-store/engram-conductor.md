# Engram 索引 ↔ Conductor

## 一、定位

本页澄清 `mooncake-store` 中两个易被混淆的命名：**Engram**（源码 `src/engram/`）与 **Conductor**（`docs/source/design/conductor/`）。经源码核实，二者是**两个独立系统**而非同一物的源码/设计别名：Engram 是 Store 之上的 embedding table 存储后端；Conductor 是 Mooncake 仓库之外、用 Go 编写的全局 KV-cache 前缀索引服务。Store 与 Conductor 的真实联系是“事件发布者 ↔ 事件消费者”。本页同时纠正 [`../00-overview/glossary.md`](../00-overview/glossary.md) 与 [`../00-overview/dependency-graph.md`](../00-overview/dependency-graph.md) 中将二者并称的表述。

## 二、目录 / 源码结构

```
# Engram（本仓库，mooncake-store 内）
mooncake-store/
├── include/engram/
│   ├── engram_store.h            EngramStore 类 (embedding 表存储后端)
│   └── engram_store_config.h     EngramStoreConfig (per-head table 尺寸)
├── src/engram/
│   └── engram_store.cpp          populate / lookup_rows 实现 (经 PyClient)
└── src/CMakeLists.txt            将 engram/engram_store.cpp 编入 store 库

mooncake-integration/store/engram_store_py.cpp   EngramStore 的 Python 绑定

# Conductor（独立 Go 项目，不在 Mooncake 主仓库内）
mooncake-conductor/conductor-ctrl/   见 docs/source/design/conductor/
├── common/ kvevent/ prefixindex/ zmq/ main.go
```

## 三、核心概念

### 1. EngramStore 是什么（源码事实）

`EngramStore`（`engram_store.h:27`）是 **Engram embedding 表的 Mooncake 存储后端**，范围刻意收窄（`engram_store.h:22-26`）：

- 调用方给定每头表尺寸 `table_vocab_sizes=[N_0…N_{H-1}]` 与行宽 `embedding_dim=D`（`engram_store_config.h:16`）；
- 为 `layer_id` 每个 head 生成一个 Store key：`engram:l{layer_id}:h{head_idx}`（`engram_store.cpp:37-40`），存 `float32` 表 `[N_h, D]`；
- `populate()` 先 `batchIsExist` 确认目标 key 为空，再 `register_buffer` + `batch_put_from` 上传，失败时回滚已写 key（`engram_store.cpp:234`）；
- `lookup_rows()` 接收 `[B,L,H]` 预计算 row id，对每个 head 构造 per-row 字节范围，经 `PyClient::get_into_ranges` 一次性物化到 `[B,L,H,D]` 输出缓冲（`engram_store.cpp:43`）。

关键点：EngramStore **不**实现 tokenizer 压缩、N-gram hashing、routing、gating 或任何模型侧 Engram 逻辑；它只是 PyClient（Store 客户端）的一个结构化使用者，把“对象→位置”映射交给底层 master/Segment，自身不做索引。设计文档 `../../docs/source/design/engram.md` 同此定义。

### 2. Conductor 是什么（设计事实）

Conductor（`../../docs/source/design/conductor/conductor-architecture-design.md`）是面向 cache-aware router 的 **KV-cache 索引器**，用 Go 实现（`mooncake-conductor/conductor-ctrl/`，独立仓库，不在本仓库源码树）：

- 订阅推理引擎或存储后端发出的 KV 事件（ZMQ SUB/DEALER），规范化为 store/remove/cleared；
- 维护 `PrefixCacheTable`：按 `ModelContext=(tenant_id, model_name, lora_name, block_size, additional_salt, instance_id)` 分域的 engine-hash → conductor-hash 映射、replica count、medium set、DP-rank set；
- 暴露 HTTP API `POST /register` / `/unregister` / `/query` / `/query_by_hash`（`../../docs/source/design/conductor/indexer-api-design.md`），返回按实例的 `longest_matched` 与按介质（`gpu`/`cpu`/`disk`）命中；
- 哈希标准为 XXH3-64 滚动序列哈希，等前缀产生等哈希直至首个差异块。

Conductor 处理的是 **token 序列前缀哈希 → 实例/介质命中** 的全局路由问题，与 EngramStore 的 embedding row 查询是两个正交维度。

### 3. 二者的真实关系

| 维度 | EngramStore | Conductor |
|------|-------------|-----------|
| 语言/位置 | C++，`mooncake-store/src/engram/`（本仓库） | Go，`mooncake-conductor/`（独立仓库） |
| 抽象层 | Store 客户端（经 `PyClient`） | 独立索引服务（HTTP + ZMQ） |
| 索引对象 | embedding row id → 字节范围（per-head 表） | prefix seq_hash → 实例/介质命中（KV block） |
| 事件角色 | 不发布也不消费 KV 事件 | **消费** Store master 发布的 KV 事件 |
| 设计文档 | `../../docs/source/design/engram.md` | `../../docs/source/design/conductor/` |

二者唯一的间接联系：Store 的 master 在 `enable_kv_events=true` 时可作为 RFC #1527 KV 事件发布者（`kv_events_bind_endpoint`），Conductor 作为订阅者把 Store 的 host/disk 池副本可见性纳入其 `/query` 响应（见 [kv-event-serialize.md](kv-event-serialize.md)）。这是“发布者 ↔ 消费者”松耦合，不是“实现 ↔ 设计”关系。

### 4. 命名混淆的来源与纠正

[`../00-overview/glossary.md`](../00-overview/glossary.md) 将 “Engram” 标注为“Store 的对象→位置索引层，对应 Conductor 设计”，[`../00-overview/dependency-graph.md`](../00-overview/dependency-graph.md) 第 4 节进一步断言 `src/engram/engram_store.cpp` 即 Conductor 索引的实现。源码核实表明该映射不成立：

- `engram_store.cpp` 的 key 模板是 `engram:l{id}:h{h}`（embedding 表），不做 token 前缀哈希、不接 ZMQ、不暴露 HTTP、无 `PrefixCacheTable`；
- Conductor 的实现语言（Go）、仓库（外部）、数据模型（rolling seq_hash）均与 `mooncake-store/src/engram/` 无源码对应。

故本 Wiki 将二者解耦叙述：EngramStore 见本页与 `engram.md`；Conductor 见 `conductor-architecture-design.md`。建议后续修订 glossary/dependency-graph 对应条目。

### 5. 应用层集成

EngramStore 的集成路径为 C++ 内核 → Python 绑定 → 应用：

- C++：`EngramStore(layer_id, config, store)` 直接持 `shared_ptr<PyClient>`（`engram_store.h:29`）；
- Python：`mooncake-integration/store/engram_store_py.cpp` 暴露 `EngramStore` / `populate` / `lookup` 等（`engram.md` 的 Python 接口契约）；
- 测试/基准：`scripts/test_engram_store.py`（可自启本地 `mooncake_master` 跑 TCP 自包含测试）与 `scripts/bench_engram_store_27b.py`。

Conductor 的集成路径则是部署期独立进程：构建 `mooncake_conductor` 二进制，静态配置 `conductor_config.json` 或动态 `POST /register` 接入各 KV 事件发布者（vLLM/SGLang/Mooncake master）。

## 四、交叉引用

- Conductor 架构与组件：`../../docs/source/design/conductor/conductor-architecture-design.md`；HTTP/事件 wire 协议：`../../docs/source/design/conductor/indexer-api-design.md`。
- EngramStore 后端契约：`../../docs/source/design/engram.md`。
- Store 作为 KV 事件发布者（Conductor 消费侧）见 [kv-event-serialize.md](kv-event-serialize.md)。
- EngramStore 经 `PyClient` 访问 Store，`PyClient` 与 Python 绑定层见 [`../08-integration/index.md`](../08-integration/index.md)。

## 五、关键源码入口

| 入口 | 定位 |
|------|------|
| `EngramStore` 类声明与职责边界 | `mooncake-store/include/engram/engram_store.h:27` |
| `EngramStoreConfig` 物理表布局 | `mooncake-store/include/engram/engram_store_config.h:16` |
| per-head key 生成 `engram:l{id}:h{h}` | `mooncake-store/src/engram/engram_store.cpp:37` |
| `populate` 上传 + 回滚 | `mooncake-store/src/engram/engram_store.cpp:234` |
| `lookup_rows_flat` 行字节范围构造 | `mooncake-store/src/engram/engram_store.cpp:43` |
| engram 编入 store 库 | `mooncake-store/src/CMakeLists.txt:76` |
| Python 绑定 | `mooncake-integration/store/engram_store_py.cpp` |
| Conductor HTTP API 契约（外部） | `../../docs/source/design/conductor/indexer-api-design.md` |
| Conductor 架构与事件流（外部） | `../../docs/source/design/conductor/conductor-architecture-design.md` |
