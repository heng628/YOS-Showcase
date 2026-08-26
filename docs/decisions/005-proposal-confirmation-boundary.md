# 005 — Proposal 是一等对象；写操作统一确认边界

- Date: 2026-08-23
- Status: **Accepted**
- 关联 Source of Truth: `docs/architecture/permission-model.md`、`agent-architecture.md`、`AGENTS.md` §30/§31

## Context

Agent 的任何写操作都必须可确认、可审计、可撤销、可追溯；若 Proposal 只是 UI 层确认卡，则无法持久、无法排队、无法过期、无法作为 Activity 与撤销依据。

## Decision

- **Proposal 是一等对象**（持久化、可排队、可过期、可审计）；Confirmation Card 只是其 UI 表现。
- 写流程统一：`Proposal → Confirmation → Execute → Result → Activity Log`。
- 确认政策 A/B/C/D + 行为等级 L0–L3（详见 permission-model.md）。
- 区分 Proposal（写动作、需确认）与 Insight/Suggestion（无动作、瞬时）。

## Alternatives

- Proposal 仅作为 UI 层确认卡（无持久对象）。
- 每种写操作各自实现确认逻辑。

## Rejected

- **仅 UI 确认卡**：无持久化与审计依据；无法表达 Accepted/Rejected/Expired/Executed 生命周期；撤销与 Activity 失去锚点。
- 各操作各自实现：确认策略碎片化，权限模型不可执行。

## Why

Proposal 是 Agent 权限体系的原子单位：把「AI 的想法」与「系统的执行」解耦，权限强制、审计、撤销全部落在一处。

## Consequences

- domain-model / database-schema 含 proposal 实体（status、riskLevel、impact、reversible 等）。
- 用户拒绝后 Agent 不得重复执行被拒动作（负向学习信号）。
- 所有执行结果入 Activity Log（Application 统一写）。
