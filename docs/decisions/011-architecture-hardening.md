# 011 — 架构加固：数据边界、离线确认与 IPC 运行时契约

- Date: 2026-08-25
- Status: **Accepted**
- 关联 Source of Truth: `AGENTS.md` §30–§32、`docs/architecture/agent-architecture.md`、`docs/architecture/permission-model.md`、ADR 003、ADR 005、ADR 010
- 适用范围：MVP 当前 Electron 本地运行时；不改变领域对象与数据库 schema

## Context

架构审查发现三个会放大故障影响的接缝：

1. DuckDB 初始化失败时，记录库可能静默降级到内存实现；SQLite 元数据仍在，但重启后记录数据会消失。
2. Proposal 确认服务和 LLM 初始化绑定；离线或未配置 LLM 时，用户无法确认已经存在的本地 Proposal。
3. IPC 的 TypeScript 类型只在编译期存在；来自 Renderer 的文件路径、LLM 配置、报告内容、手动录入和 Agent 参数缺少统一的运行时约束。

## Decision

### 1. 发布态数据边界 fail-fast

- `createRecordStore` 增加 `fallbackToMemory` 显式开关。
- 桌面发布路径使用 `fallbackToMemory: false`；DuckDB 初始化失败直接抛错，由主进程的 `health.initError` 展示。
- DuckDB `open()` 先创建并关闭内存实例探针，尽早验证 native runtime，而不是等到首次读写才暴露错误。
- 内存实现保留给测试和明确的开发/兼容场景；默认行为仍保持测试兼容，但发布组合根必须显式关闭降级。

### 2. Proposal 确认与 LLM 解耦

- `ConfirmUnderstandUseCase` 在本地数据边界初始化后立即创建。
- LLM 只负责理解、分析和生成 Proposal；确认、执行和 Activity Log 属于本地 Application 能力。
- 未配置 LLM 或离线时，已存在的 Proposal 仍可被用户确认和执行。

### 3. IPC 入口增加运行时契约

- 新增 `apps/desktop/src/main/ipc-validation.ts`，主进程把来自 Renderer 的高风险参数视为 `unknown` 后再校验。
- 当前覆盖：文件导入、报告导出、LLM 配置、手动资产创建、手动行录入、Agent 命令。
- 校验内容包括对象结构、字符串长度、空字符、文件扩展名、URL 协议、原子字段类型、数组/单元格数量和 Agent 文本大小。
- 校验失败返回结构化错误，不进入 UseCase、文件系统或 LLM。

## Alternatives

- 继续依赖 TypeScript 类型：开发期方便，但无法约束被注入或错误调用的 Renderer payload。
- DuckDB 失败时继续静默内存降级：启动看似成功，但会造成数据不可见且重启丢失。
- 让确认流程复用 LLM 初始化：实现简单，但违反本地权威和离线可用原则。

## Rejected

- **生产态静默降级**：拒绝，因为“数据看似存在、重启后消失”比启动失败更危险。
- **用 IPC 校验替代 Application 校验**：拒绝；IPC 校验只负责边界形状和资源上限，业务权限、owner scope、引用和状态机仍必须由 Application/Domain 负责。
- **本次直接重做 main/Application 分层**：拒绝；这是下一阶段结构性重构，需先定义桌面端 Application Facade 和统一错误协议，避免一次性扩大回归面。

## Consequences

- 主进程启动更诚实：DuckDB native 或目录不可用时显示初始化错误，不再悄悄进入内存记录库。
- 用户可以在离线状态完成本地 Proposal 确认；LLM 能力仍会按现有规则提示需要配置/联网。
- IPC 校验模块可单独测试，后续新增 channel 应沿用同一模式。
- `main/index.ts` 仍是组合根且仍有部分 Repository 直连；Application Facade 未在本次完成。
- SQLite 与多个 DuckDB 文件之间仍没有跨存储事务/恢复协议；完整备份、恢复和 Dashboard 布局持久化仍属于后续任务。

## Verification

- `pnpm --filter @yos/database typecheck` ✅
- `pnpm --filter @yos/desktop typecheck` ✅
- `pnpm test -- packages/database/src/index.test.ts apps/desktop/src/main/ipc-validation.test.ts` ✅（12 tests）
- 新增数据库测试覆盖：显式关闭 DuckDB → 内存降级时必须抛错。
- 新增 IPC 测试覆盖：合法 payload 通过，非法扩展名、协议、类型、数量和空文本被拒绝。
