# 004 — 三轴状态模型（Lifecycle × Execution × Health）

- Date: 2026-08-23
- Status: **Accepted**
- 关联 Source of Truth: `docs/architecture/state-model.md`、`AGENTS.md` §30

## Context

对象在运行中同时具有「存在生命周期」「异步执行进度」「数据健康」三种互相独立的维度；扁平枚举必然混淆（如把数据滞后 Stale 当成执行失败 Failed）。

## Decision

三轴正交：
- Lifecycle：`Draft → Active → Archived → Deleted`（受确认门槛对象加 `Proposed`）
- Execution：`Queued → Running → Paused → Completed / Failed / Cancelled`
- Health：`Healthy / Warning / Error / Stale / Disconnected`
每对象只在需要的轴开槽；语义边界：`Empty ≠ Draft`、`Muted ≠ Paused ≠ Deleted`、`Stale`(Health) ≠ `Failed`(Execution)。

## Alternatives

- 单一扁平状态枚举。
- 按对象各定一套独立枚举。

## Rejected

- **扁平枚举**：无法表达「执行失败的资产同时健康」「暂停的监测仍在监控」等组合；语义污染。
- 各对象独立枚举：失去统一守卫与一致性，接口层复杂。

## Why

三个维度本质正交；正交建模表达力最强、守卫最统一，也直接支撑用户可见「人话状态」（内部微状态不外泄）。

## Consequences

- schema 用 `lifecycle` / `execution` / `health` 三列（不是一列）。
- Proposal 的 PendingConfirmation 不入目标对象生命周期（对象停 Draft/Proposed）。
- 删除默认软删 + 引用检查 + 降级，不制造悬空引用。
