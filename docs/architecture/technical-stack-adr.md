# 技术选型 ADR（Technical Stack）

- 状态：**已落地基线**（初始选型说明保留；当前实现差异见 §2.1）
- 日期：2026-08
- 适用范围：MVP（本地优先数据 + 云端智能/能力面）
- 前置决策：`AGENTS.md` §19/§26/§27/§30/§31；Hybrid 架构 ADR（对话已确认）

---

## 1. Context

本节保留初始选型背景。当前实现已经落地 Electron + TypeScript monorepo、SQLite catalog、DuckDB RecordStore、React Renderer 和 OpenAI 兼容 LLM 边界；产品仍遵循 Hybrid（本地优先数据 + 云端智能/能力面），MVP 不实现云同步、Team/Enterprise、API/OAuth、多来源聚合。

本 ADR 把上述产品决策转化为**可执行的工程选型**。原则：

> 能简单就简单；能本地就本地；能抽象接口就不提前实现；能延后到 V1 就不要放进 MVP。

---

## 2. Decision（选型结论）

| 项 | 选择 |
|---|---|
| Desktop Runtime | **Electron**（main/preload/renderer；`contextIsolation: true`，DB 只在 main） |
| Language | **TypeScript（strict）全栈统一**，无 Rust 要求 |
| Frontend | **React 19 + Vite**；MVP 使用组件/页面本地 React state；Renderer 经 **preload IPC** 访问能力 |
| Local Database | **SQLite（catalog/实体/配置） + DuckDB（原始记录 + 确定性 Analytics）** |
| ORM / DB Access | **better-sqlite3 + Drizzle ORM**（SQLite）；DuckDB 用 **duckdb 官方 Node 绑定 + 类型化 SQL 封装**（无 ORM） |
| Monorepo | **pnpm workspaces + Turborepo** |
| UI / Design System | CSS Variables + 自建组件；`packages/ui` 提供共享 UI；当前 MVP 未引入 Tailwind/Radix/ECharts |
| AI Client | 自建薄抽象 **`LLMClient` 接口**（`packages/ai`），首个实现为 **OpenAI 兼容 Provider**；结构化输出；**用量计量在 client 边界** |
| Testing | **Vitest（unit + integration，内存 SQLite/DuckDB）+ Playwright（最小 E2E，Electron）** |
| Lint/Format/Type/CI | **ESLint(flat) + Prettier + tsc --noEmit + GitHub Actions**（lint/typecheck/test） |

### 2.1 Current Implementation Alignment

- React 实际版本为 19；当前没有 Zustand、Tailwind、Radix 或 ECharts 依赖，Renderer 以本地 React state 和现有 CSS token 为主。
- `packages/database` 对发布态记录库使用 DuckDB fail-fast；内存 RecordStore 仅作为显式兼容/测试路径。
- Proposal 确认是本地 Application 能力，不依赖 LLM 是否配置。
- IPC 高风险输入在 `apps/desktop/src/main/ipc-validation.ts` 进行运行时校验。
- 后续若引入全局状态、组件原语或图表库，必须另行评估依赖成本并更新本 ADR。

---

## 3. Alternatives（逐项比较）

### 3.1 Desktop Runtime：Electron vs Tauri

| | Electron | Tauri |
|---|---|---|
| 运行时 | Chromium + Node.js | 系统 WebView + Rust core |
| 体积/内存 | 大（~100MB+） | 小（~10MB） |
| Node 能力 | 完整（文件、子进程、原生模块、后台任务） | 无 Node；后端走 Rust command |
| 本地 DB/分析 | main 进程直接跑 better-sqlite3 / duckdb（Node 绑定） | 需 tauri-plugin-sql（Rust 侧 SQLite）或 WASM，DuckDB 要侧车/plugin |
| 全栈语言 | **纯 TypeScript** | TS + Rust（或牺牲能力） |
| 生态/迭代速度 | 最成熟、迭代最快 | 较新、插件生态小 |

**为什么选 Electron**：MVP 的核心是本地数据权威 + 本地确定性分析 + AI 客户端 + 后台调度，这些全部在 Node 生态里直接可用（better-sqlite3 / duckdb / 文件解析 / IPC）；全栈 TS 单一语言，迭代速度最快。Tauri 的「小体积/低内存」不是 MVP 的瓶颈。

**Tauri 何时再评估**：V1+ 若体积/内存成为获客差异点，或安全审计要求更小攻击面时，可迁移（领域/应用/分析层全是纯 TS，可复用；仅 main 侧壳与 DB 宿主重做）。

### 3.2 本地数据库：SQLite vs IndexedDB vs DuckDB vs 只用其一

| | SQLite | IndexedDB | DuckDB | 只用其一 |
|---|---|---|---|---|
| 实体/目录/配置 CRUD | ✅ 强 | ⚠️ 弱 | ⚠️ 弱（单写者、DML 弱） | — |
| Excel/CSV 导入 | 需自己解析写入 | ❌ | ✅ 原生 ingest | — |
| 聚合/分析性能 | 中 | ❌ | ✅ 列式 OLAP，强 | — |
| 版本/软删/事务/FK | ✅ 强 | ⚠️ | ⚠️ 追加型、无 FK | — |
| 未来云同步 | 容易（行级 id/version） | 差 | 中（文件级/分区） | — |

**为什么选「SQLite + DuckDB 分工」**：
- **SQLite = 系统权威存储**：catalog 与配置（User/Space/Workspace/DataAsset 元数据/Field/Metric/Mount/Module/Monitoring/Report/Proposal/ActivityLog/AgentTask），行级事务、FK、软删、版本、同步缝，全部自然。
- **DuckDB = 原始记录 + Analytics 引擎**：按 DataAsset 一个库文件（或分区）存原始记录，追加/版本语义天然；CSV/Excel 原生导入；确定性指标（GMV/ROI/MoM/YoY…）用 SQL 聚合在本地跑——正好落实「确定性计算由本地程序完成，不交给 LLM」。
- **IndexedDB 出局**：不是权威存储的合理载体，无 SQL、难版本化、难同步、分析性能差。
- **只用其一**：SQLite-only 会拖慢 Analytics；DuckDB-only 做不了系统目录 CRUD。

### 3.3 ORM / DB Access

- **Drizzle ORM + better-sqlite3**：TS-first、类型安全 schema、轻量、迁移工具（drizzle-kit）、无运行时引擎二进制（对比 Prisma 在 Electron 打包更干净）。**选中。**
- Prisma：引擎二进制 + 生成步骤在 Electron 打包中额外复杂度，且较重——不选。
- Kysely：优秀的类型安全查询构建器，作为备选；Drizzle 的 schema 定义方式与迁移体验更直接——不选（可再评估）。
- DuckDB 侧：用 `duckdb` Node 绑定 + 内部类型化 SQL 封装（`RecordStore` 读接口），**不引入 ORM**（OLAP 场景 ORM 无收益）。

### 3.4 Monorepo

- **pnpm + Turborepo**：pnpm 的严格 node_modules 隔离**天然帮助强制包依赖边界**（UI/Agent 无法隐式 import 到 database）；Turborepo 只做任务缓存，轻。**选中。**
- Nx：功能强但重、概念多，MVP 不需要——不选。
- npm/yarn workspaces：可行但依赖边界松散——不选。

### 3.5 UI / Design System

- **Tailwind（token 即 CSS Variables）+ Radix 原语 + 自建组件 + ECharts**：Tailwind 配置直接承载 Design Tokens（Primitive→Role→Component 三层可映射）；CSS 变量支持未来 Theme JSON 整体替换；Radix 提供可访问的 Modal/Popover/Command 等原语；不引入 MUI/Ant 这类与「AI Native OS 克制视觉」冲突的重组件库。ECharts 满足 Line/Bar/Area 的轴/tooltip/对比/异常标注全控制。**选中。**
- 备选说明：Recharts（更轻但控制力弱）；Svelte/Vue（生态与招人/Agent 成熟度不如 React）。

### 3.6 AI Client

- 自建 **`LLMClient` 接口**（`complete / completeStructured`），**Domain 永不 import 具体厂商**；首个实现 OpenAI 兼容 Provider（可指向任意兼容端点）；**用量计量（tokens/cost）在 client 边界统一拦截**。**选中。**
- 不直接把 SDK 撒进业务代码；不在 MVP 接本地模型（预留接口即可）。

### 3.7 Testing / Quality / CI

- **Vitest**（快、TS 原生）做 unit + integration（SQLite in-memory / DuckDB in-memory）；**Playwright** 只覆盖关键 vertical slice 的 Electron E2E。
- **ESLint flat + Prettier + tsc --noEmit**；CI = GitHub Actions 三任务（lint / typecheck / test）。
- 不引入复杂测试矩阵、不建 e2e 全覆盖（MVP 不必要）。

---

## 4. Why（选择理由总述）

1. **单一语言 TS 全栈**：领域/应用/数据/分析/AI/UI 全部 TypeScript，Agent 生态、类型安全、迭代速度最优。
2. **本地权威的合理载体**：SQLite（系统目录）+ DuckDB（记录与分析）精确匹配「本地权威 + 确定性分析」。
3. **边界可强制**：pnpm + 包边界规则 + Electron contextIsolation + preload IPC，把「UI/Agent 不直连 DB、Domain 不依赖 UI/Agent/LLM」从口号变成工程约束。
4. **AI 不入侵 Domain**：薄 `LLMClient` 接口 + 边界计量，Domain 无 LLM 依赖。
5. **不过度设计**：云同步/OAuth/Team/多租户/计费只留接口缝，不进 MVP。

---

## 5. Trade-offs（代价）

- Electron 体积大、内存高 → 接受，MVP 迭代优先；Tauri 作为未来迁移选项（领域层 TS 可复用）。
- 双数据库（SQLite+DuckDB）比单库多一个概念 → 换取「目录 CRUD + OLAP 分析」各自最优，代价可控（DuckDB 只在 RecordStore/Analytics 出现）。
- better-sqlite3 是原生模块 → 需要 electron-rebuild 流程（一次性配置）。
- OpenAI 兼容 Provider 是首个实现 → 若未来接本地模型/多厂商，走同一接口扩展，不侵入业务。

---

## 6. MVP Scope（本 ADR 范围内）

**做**：Electron+TS+React 工程骨架；SQLite 目录 + DuckDB 记录/分析；Excel/CSV 导入；DataAsset/Field/Metric/Workspace/Space/Mount/Module；Proposal/Confirmation/ActivityLog；AI 读/算/理解/提案（云 LLM）；本地轻调度；AI 用量计量；薄云能力面（身份 stub + LLM + 计量）。

**只预留接口（不实现）**：云同步、OAuth、API Connector、Team/Enterprise/多租户、真实计费、多来源聚合、Schema 对齐、云端自主 Agent、多人实时协作、Theme Editor/Marketplace、本地模型。

**离线降级（Hybrid 硬规则）**：本地数据与确定性分析可完全离线工作；依赖云 LLM 的能力（语义理解/洞察/提案）在无网络时明确提示「需要联网」。原始数据**绝不默认上传云端**。

**第一条 Vertical Slice（MVP 唯一完整闭环，不扩展）**：Excel/CSV 导入 → DataAsset(Draft) → 字段识别/类型推断/数据体检 → AI 语义理解 → 字段映射 Proposal + Metric Proposal → 用户确认 → DataAsset Active → Space/Workspace/Mount → Metric → Module → AI 只读分析 → 一个 Monitoring Proposal → Confirmation Card → 用户确认 → Application Execute → Activity Log。

---

## 7. Future Migration Path

- **Tauri 迁移**：若体积/内存成为产品差异点 → 领域/应用/分析/UI 包原样复用，仅替换 Electron 壳与 DB 宿主（SQLite/DuckDB 走 Rust 侧或 WASM）。
- **云同步（V1）**：catalog 表已带稳定 UUID id + version + owner/scope + soft-delete，同步协议在此基础上做（无需改 schema 拓扑）。
- **云端自主 Agent（V1）**：Application/Tool/Domain 与本地共用，把「自主运行时」部署到云端，读写经授权作用域/快照。
- **Team/Enterprise（V2）**：owner/scope 泛化（User→Team/Org）+ Membership/role + 隔离，沿用「资产=顶层共享」拓扑，不重做数据模型。
- **AI 多厂商/本地模型**：`LLMClient` 接口不变，新增 Provider 实现。
