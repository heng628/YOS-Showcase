# YOS Agent 记忆库落地技术调研报告

> 目的：为 YOS（Electron + SQLite(better-sqlite3+Drizzle) + DuckDB、离线优先、云端仅 LLM API）设计「Agent 记忆库 = 结构化事实（偏好/指标语义/规则/历史决策）+ 可能的部分语义检索」提供落地技术依据。
> 结论倾向：**本地优先、体积可控、不臃肿、离线可用；嵌入可云端顺带生成或本地小模型。**
> 调研时间：基于 2025 年末至 2026 年初公开资料。凡「维护状态」标注均据公开仓库/Issue/发布记录，非臆测。

---

## 一、本地向量检索方案对比

**sqlite-vec**（SQLite 扩展，向量存进你的 SQLite 文件）
- 原仓库由 Alex Garcia（asg017）维护，但自 2024 下半年后基本停滞，社区已有「sqlite-vec is, at least temporarily, dead」的讨论 [haiku.rag#41](https://github.com/ggozad/haiku.rag/issues/41)。
- 活跃续作在延续此代码库：**[stacklok/sqlite-vec](https://github.com/stacklok/sqlite-vec)**（Stacklok 承接，持续修复与发布）与 **[photostructure/sqlite-vec](https://github.com/photostructure/sqlite-vec)**（社区 fork，新增距离约束、分页、`optimize` 回收空间）。
- Node 侧有封装：`@photostructure/sqlite-vec`、`@tan-yong-sheng/sqlite-vec-wasm-node`（wasm，免原生编译）[npm](https://www.npmjs.com/package/@photostructure/sqlite-vec)。
- **优点**：与你已有 SQLite 零额外文件、事务/备份/加密同一套；体积极小；与 Drizzle 共存自然。
- **劣势**：原 API 成熟度一般；构建需载入扩展二进制（`better-sqlite3` 可 `loadExtension`）；性能对百万级向量为「够用」，大库需外部索引管理。

**DuckDB VSS（官方扩展）** [duckdb-vss](https://github.com/duckdb/duckdb-vss) / [docs](https://duckdb.org.cn/docs/stable/core_extensions/vss)
- HNSW 索引，官方扩展、随 DuckDB 演进（扩展 ABI 有向前/向后兼容讨论，需关注版本匹配 [duckdb#13894](https://github.com/duckdb/duckdb/pull/13894)）。
- **优点**：与已有 DuckDB 天然同源，SQL 内做向量+HNSW，适合「数据分析 + 向量」一体；Node 经 `@duckdb/node-api` 可加载。
- **劣势**：VSS 相对较新、API 仍偏 alpha；索引需显式构建/刷新；Embedding 写入语法的版本敏感性高。适合作为「查询/分析型」增强，不太适合作为高频小体量记忆主存储。

**LanceDB（嵌入式，Lance 列式格式）** [LanceDB](https://github.com/lancedb/lancedb) / [npm](https://newreleases.io/project/github/lancedb/lancedb/release/v0.31.0-beta.0)
- Apache 2.0，嵌入式（不像 Qdrant/Weaviate 需要整套服务态），原生 Rust 绑定；有 JS 客户端与常被高频迭代的 beta 版本（如 v0.31.0-beta.0）。
- **优点**：多模态、为大规模向量/文件设计，性能与扩展性最稳，官方提供「离线/Local 模式」。
- **劣势**：需引入**另一套存储格式 + 原生 `.node` 绑定**，会加大 Electron 体积与打包复杂度（rebuild/ABI/跨平台）；与 SQLite 无共享关系，需额外管理数据一致性。对「记忆库」这种小体量有些「杀鸡用牛刀」。

**FAISS**（`@faiss-node/native`、`@vectorsearch/faiss`）[yarn/@faiss-node/native](https://classic.yarnpkg.com/en/package/@faiss-node/native) / [socket/@vectorsearch/faiss](https://socket.dev/npm/package/%40vectorsearch%2Ffaiss)
- 算法最成熟（HNSW/IVF），但**纯原生 C++**：Electron 下 build/rebuild 与跨平台分发痛，且 `faiss` 本身**不带持久化/元数据/关系**，需自己管理索引文件与关联表，等于重造管理系统。

**Chroma / embed-js**（[Chroma JS Client v3](https://www.trychroma.com/changelog/js-client-v3) / [chromadb](https://github.com/chroma-core/chroma)）
- 语义检索 API 友好、有 JS 客户端；但 Chroma 常依赖**内嵌或子进程的 DuckDB/Rust 存储**，JS 端要么起 server、要么内部带较重依赖。
- JS 生态里 `embed-js` 更像「文档嵌入工具链」，不是主存。属「开发快但较臃肿」路线。

**总结（与 SQLite 共存成本）**：若记忆为**小/中体量、强结构化 + 部分语义**，`sqlite-vec`（用社区活跃 fork）+ FTS5 是**成本最低、与你现状最贴合**；DuckDB VSS 适合同时做向量分析；LanceDB 适合「大规模/多模态 + 可接受更大体积」；FAISS/Chroma 因原生/臃肿在 Electron 小体量场景性价比低。

---

## 二、本地 embedding 能力

**Node 端可用路径**：**[transformers.js](https://github.com/huggingface/transformers.js)**（Xenova，现归 HuggingFace，包名 `@huggingface/transformers`）基于 **ONNX Runtime**（`onnxruntime-node`/`onnxruntime-web`），可在 Node/Electron 用 `pipeline('feature-extraction', model)` 直接产出向量 [DataStax Node.js 教程](https://www.datastax.com/blog/how-to-create-vector-embeddings-in-node-js)。

**模型选择（质量/体积/速度权衡，体积为量化 ONNX 约数，勿当精值）**：
- `all-MiniLM-L6-v2`：**约 20–25 MB（量化 ONNX）**、384 维。速度最快、体积最小，质量中上，适合「记忆/事实碎片」这类短文本。
- `bge-small-en-v1.5`：约 30–35 MB（量化）、384 维，质量更高，长文本稍好。
- `GIST-small-Embedding-v0`（[Xenova 量化版](https://huggingface.co/Xenova/GIST-small-Embedding-v0)）、`gte-small`：同档小模型，作为备选。
- `nomic-embed-text-v1.5`：约 250+ MB（fp32/非量化），质量最好但对小体积不友好。

**云 vs 本地取舍**：
- **云 embedding**：质量高（OpenAI text-embedding-3-small 等）、零本地体积；成本随调用；隐私/离线受限。**契合点**：调用 LLM 时「顺带」生成嵌入，几乎零额外成本与请求。
- **本地小模型**：离线可用、隐私、零 API 成本；代价是体积（20–35 MB/模型）+ 首次加载/推理耗时，且小模型对「中文/领域术语」质量弱于云端大模型。

**工程要点**：
- 采用**混合策略**：在线时用云 embedding（顺带生成、质量高），离线时降级到本地 MiniLM/bge-small（离线优先，缺省语义检索可用性）。
- 本地模型**懒加载 + 常驻管线**，避免每次调用冷启动；量化 ONNX 已内置（transformers.js 加载 `quantized: true`）。
- 嵌入**维度要固定**：云/本地切换时若维度不同，需二者在**同一列/同表**统一，或各建索引后 union，避免维度不一致。

---

## 三、结构化记忆存储

**SQLite 关系表 + FTS5 是否够用？**
- 对「偏好/指标语义/规则/历史决策」这类**碎片化事实**，关系表（`memories`、`preferences`、`decisions`，含 type/source/ts/confidence/valid_until）即够。
- **FTS5**（SQLite 内置，成熟稳定）做**关键词/规则匹配**很可靠；要语义就叠加列式向量（如 sqlite-vec 的 `vec0` 虚拟表 / 或用 DuckDB VSS）。**关系表 + FTS5 = 结构化记忆 + 确定性检索**，是「最小可落地」的核心。
- 何时只需关系表：记忆为**可枚举事实/枚举型偏好**（如时区、单位、默认口径），无需语义。何时需要向量：**自由文本洞察、近似语义召回**（如「上次我们怎么处理这个指标异常」）。

**知识图谱先例（本地）**：
- **[LiteGraph](https://github.com/microsoft/litegraph)**（Semantic Kernel 的图记忆数据模型）与 SK 的 **Neo4j/关系图存储**（[SK 文档 neo4j-memory](https://github.com/MicrosoftDocs/semantic-kernel-docs/blob/main/agent-framework/integrations/neo4j-memory.md)）演示了「实体+关系」图谱——但成熟生态多在 C#/Python，且大多走 Neo4j/Qdrant（较重服务态）。
- **本地关系表表达图**：用 `memories(id, subject, predicate, object, ts)` 三元组（RDF 风格）存入 SQLite 即可支持简单图查询与「事实间关联」；或做 **DuckDB 的 `list`/递归**。**何时需要真图谱**：当关系规模大、需多跳推理（「X 的依赖链上是谁」）时才引入；**关系表多数情况足够**，先不引入图。

---

## 四、Electron 端落地方案先例

- **[Obsidian Copilot](https://github.com/logancyang/obsidian-copilot)**：基于 Obsidian vault（Markdown 即本地存储），聊天/记忆/知识检索时构建向量索引（Chroma/本地嵌入），记忆写入 Markdown + 向量化 [memory-design.md](https://github.com/logancyang/obsidian-copilot/blob/632c1e81/src/memory/memory-design.md)。**教训/模式**：用「宿主已有存储（vault）+ 外来向量索引」，零额外主存；但 Markdown 检索性能有限、需单独索引层。这与 YOS「用已有 SQLite/DuckDB 承载」思路一致。
- **[AnythingLLM](https://github.com/Mintplex-Labs/anything-llm)**：Node 应用，**向量库可插拔**（多为 LanceDB/Chroma），元数据与内存状态用 SQLite（Prisma），形成「元数据(关系) + 向量(LanceDB)」双库模式 [Vector Database Architecture](https://deepwiki.com/Mintplex-Labs/anything-llm/6.1-vector-database-architecture)。**教训**：如果走双库，要做到**可插拔+抽象**，避免绑死；LanceDB 提供良好离线嵌入能力。
- **[localGPT](https://github.com/jaechoon2/localGPT)（与 tracegyx/localGPT）**：100% 本地，用本地 embedding + 本地 LLM + Chroma 向量库。**教训**：隐私/离线是卖点，但全靠本地小模型 + 外部向量库，体积与启动成本高；并非默认推荐为轻量记忆库。

---

## 五、建议结论（YOS 推荐栈）

**核心判断**：YOS 已有 SQLite 与 DuckDB，记忆库体量小、价值高。**不要在「记忆库」上加第二套重型存储**；优先「SQLite 关系 + FTS5 + 轻量向量」，最大化复用现有权威数据底座（符合 ADR「本地优先 / 数据本地」与「避免不必要复杂度」原则）。

**推荐栈（主推）**
- 存储：**SQLite（沿用 better-sqlite3 + Drizzle）** 作为记忆唯一主存；表设计 `memories(id, kind[preference|fact|rule|decision], subject, predicate, object, value, source, confidence, valid_until, ts)` + `memory_links(subject, predicate, object, ts)` 表达简单图/RDF 关系。
- 检索：**FTS5（确定性/关键词/规则）为底座** + `sqlite-vec`（用**社区活跃 fork**：stacklok/sqlite-vec 或 photostructure/sqlite-vec）的 `vec0` 虚拟表做语义召回；若后期需把向量的分析能力并入数据侧，再补 **DuckDB VSS**（同一 DuckDB，零新文件）。
- Embedding 策略：**混合**。在线调用 LLM 时**顺带**用云 embedding（高质量、近零额外成本）；离线**降级**到本地 `all-MiniLM-L6-v2`（约 20–25 MB 量化 ONNX）或 `bge-small-en-v1.5`，经 transformers.js 常驻管线懒加载。维度统一策略要提前设计（本地小模型与云模型不同维时，或用「同一列两套索引 + 检索时分别召回合并」）。

**两档方案**
- **最小可落地（MVP）**：`SQLite 关系表 + FTS5` 为主，向量检索**暂不上**或仅在**在线**时用云 embedding + sqlite-vec 的轻量列。体积 ≈ 零新增（除几十 MB 的 ML 库）；离线时降级为「元数据 + 关键词」检索，保证永远可用。
- **完整版（V1+）**：`SQLite(关系) + FTS5 + sqlite-vec(向量，活跃 fork)`，在线用云嵌入、离线用本地 MiniLM；到「大规模多模态记忆 + 需多跳图谱推理」时，再评估 **LanceDB**（多模态/大规模）或 **DuckDB VSS**（向量+分析一体）。知识图谱仅在确实需要多跳关联时引入关系三元组扩展，不必上 Neo4j/Qdrant。

**已知风险**：sqlite-vec 原仓库停滞、必须锚定活跃 fork；DuckDB VSS 版本/ABI 敏感性；本地 embedding 对中文与领域术语质量弱于云模型；LanceDB/FAISS 的原生绑定会显著增加 Electron 体积与打包复杂度——在当前小体量记忆场景下**不建议**为首选。

---

## 来源清单

- sqlite-vec 停滞讨论：https://github.com/ggozad/haiku.rag/issues/41
- 社区活跃 fork：https://github.com/stacklok/sqlite-vec ｜ https://github.com/photostructure/sqlite-vec
- sqlite-vec npm 封装：https://www.npmjs.com/package/@photostructure/sqlite-vec ｜ https://www.npmjs.com/package/@tan-yong-sheng/sqlite-vec-wasm-node
- DuckDB VSS：https://github.com/duckdb/duckdb-vss ｜ https://duckdb.org.cn/docs/stable/core_extensions/vss ｜ ABI 兼容 https://github.com/duckdb/duckdb/pull/13894
- LanceDB：https://github.com/lancedb/lancedb ｜ LanceDB-js 版本 https://newreleases.io/project/github/lancedb/lancedb/release/v0.31.0-beta.0
- FAISS Node 绑定：https://classic.yarnpkg.com/en/package/@faiss-node/native ｜ https://socket.dev/npm/package/%40vectorsearch%2Ffaiss
- Chroma JS Client v3：https://www.trychroma.com/changelog/js-client-v3 ｜ https://github.com/chroma-core/chroma
- JS 客户端向量库选型参考：https://duan.li/blog/choosing-a-vector-database-for-client-side-rag-in-javascript ｜ http://www2.sqlite.org/forum/forumpost/02b3aa51b1
- transformers.js：https://github.com/huggingface/transformers.js ｜ Node 嵌入教程 https://www.datastax.com/blog/how-to-create-vector-embeddings-in-node-js
- 小模型量化 ONNX：https://huggingface.co/Xenova/GIST-small-Embedding-v0
- LiteGraph（语义内核图记忆数据模型）：https://github.com/microsoft/litegraph ｜ SK 图记忆 https://github.com/MicrosoftDocs/semantic-kernel-docs/blob/main/agent-framework/integrations/neo4j-memory.md
- hybrid SQLite+LanceDB+FTS5 先例：https://github.com/tenfourty/kbx
- Obsidian Copilot 记忆设计：https://github.com/logancyang/obsidian-copilot/blob/632c1e81/src/memory/memory-design.md ｜ https://github.com/logancyang/obsidian-copilot
- AnythingLLM 向量库架构：https://github.com/Mintplex-Labs/anything-llm ｜ https://deepwiki.com/Mintplex-Labs/anything-llm/6.1-vector-database-architecture
- localGPT（本地/离线）：https://github.com/jaechoon2/localGPT
