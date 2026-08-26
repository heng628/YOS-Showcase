# 006 — Metric 是用户级知识资产（不属于 DataAsset）

- Date: 2026-08-23
- Status: **Accepted**
- 关联 Source of Truth: `docs/architecture/domain-model.md`、`AGENTS.md` §02/§04（已修正）

## Context

指标既可能是单资产聚合（GMV），也可能跨资产（Profit = A.收入 − B.成本）；若归属某个 DataAsset，跨资产指标无处安放，且同一指标会重复定义。

## Decision

`User 1─N Metric`；Metric 通过 `metric_input` 引用 ≥1 个 Field（可跨资产）；`dataScope(single/cross)`、`origin(system/user/ai)`、`status(proposed/active)` 是**属性**而非类型。
- Formula 归 Metric 定义；Relation 是独立关系对象（V1）；**Field 只含原子值类型**。

## Alternatives

- A Metric 属于 DataAsset。
- C 拆成 DataAsset Metric + Global Metric 两型。

## Rejected

- **A（属于 DataAsset）**：跨资产指标（Profit/ROI 跨表）无法归属；同一业务指标重复定义，违背「定义只有一份」。
- **C（两型分裂）**：分裂脑——绑定/复用要兼容两型，易产生副本，违背唯一性。

## Why

指标是「用户的分析知识资产」，随用户沉淀、跨工作台复用；属性化（而非类型化）保留「按资产看指标」的过滤视图，同时不引入分裂。

## Consequences

- schema：metric 无 data_asset_id；metric_input(Metric N─M Field) 承载字段引用。
- 可用性规则：工作台内 Metric 可用 ⟺ 其引用数据集均已 Mount。
- 删除 Metric = 软删 + 引用预告 + 引用 Module 降级。
- AGENTS.md §04 已把 Formula/Relation 移出 Field 类型。
