# 002 — DataAsset 是用户级顶层共享资产（不属于 Workspace）

- Date: 2026-08-23
- Status: **Accepted**
- 关联 Source of Truth: `docs/architecture/domain-model.md`、`database-schema.md`、`AGENTS.md` §02（已修正）

## Context

原始层级图把 DataAsset 画在 Workspace 之下（线性树），但同时要求「一个 Data Asset 可被多个 Workspace 使用、不得复制」。线性树与 N:M 共享无法同时成立。

## Decision

`User 1─N DataAsset`（顶层共享）；`Workspace 1─N Mount N─1 DataAsset`（工作台经 Mount 使用，不拥有）；原始数据只存一份。数据模型上 `data_asset` 表**没有 workspace_id**；层级图仅作导航理解。

## Alternatives

- A 引用关系模型（薄 N:M 关联）
- B 挂载/视图模型（Mount 作为一级对象承载工作台侧使用配置）← 最终采纳
- C 分裂变体（资产归属方案的两型分裂）

## Rejected

- DataAsset 归属某个 Workspace（无法共享、会复制数据）。
- 薄 N:M 关联（无一级 Mount）：无法承载工作台侧差异化使用配置，后续必然升级成实体（返工）。

## Why

DataAsset 是「资产」，Workspace 是「使用环境」；归属必须落 User。Mount 作为一级对象精确承载「工作台如何使用资产」，且为 Team/Enterprise 权限预留更细授权落点（授权 Mount/视图）。

## Consequences

- schema 中 data_asset 无 workspace_id；mount 为连接表并承载 alias/scope_filter。
- 删除 Workspace 不删资产；删除资产需引用预告 + 降级。
- 后续 Team/Enterprise 只需把 owner 从 User 泛化为 Team/Org，拓扑不变。
