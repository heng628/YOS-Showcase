# 003 — Agent 永不直连 DB / Repository / 文件写入口

- Date: 2026-08-23
- Status: **Accepted**
- 关联 Source of Truth: `docs/architecture/agent-architecture.md`、`permission-model.md`、`AGENTS.md` §19/§31

## Context

Agent 是智能执行层，但 LLM 非确定性；一旦 Agent 可直连 DB，幻觉/误操作即可改写或删除原始数据，并绕过确认与审计。

## Decision

调用链强制：`Agent → Tool Layer → Application Service → Permission/Confirmation 闸门 → Domain → Repository → Data`。
- 工具白名单、typed（name/purpose/inputSchema/outputSchema/riskLevel/access/requiresConfirmation/allowedActor）。
- **执行类（execute）工具不对 Agent 开放**：Agent 的写能力止步于 propose。
- 外部访问必须经 Connector/Sync Service + 用户授权 scope + License/Entitlement。

## Alternatives

- Agent 直连 DB/Repository。
- 只做运行时约定、不做工程强制。

## Rejected

- **Agent → DB / Repository / 任意文件系统写入口**：绕过权限/确认/审计；LLM 幻觉可造成数据破坏；违反「原始数据永不自动执行」。
- 只做口头约定：无法防止依赖漂移与未来违反。

## Why

中间层（Tool Layer + Application Service）是权限、确认、审计、事务、软删/版本的强制插入点；直连会全部失效。

## Consequences

- `packages/agent` 不依赖 `packages/database`（pnpm 隔离 + ARCHITECTURE GUARD + 测试断言三重保证）。
- Electron 中 DB 只在 main；renderer 只经 preload 最小 typed API。
- 写操作统一 `Proposal → Confirmation → Execute → Result → Activity Log`。
