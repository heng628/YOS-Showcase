# 001 — Hybrid：本地优先数据 + 云端智能/能力面

- Date: 2026-08-23
- Status: **Accepted**
- 关联 Source of Truth: `docs/architecture/technical-stack-adr.md`、`AGENTS.md` §19

## Context

产品同时需要：(a) 用户拥有并沉淀自己的数据（离线、隐私、可迁移、不锁数据）；(b) AI 理解/解释/提案（需 LLM）；(c) 团队/企业、跨设备、商业化计量；(d) 关掉 App 后仍要运行的自主任务（监测/定时分析/报告/同步）。

## Decision

**Hybrid = 本地优先数据 + 云端智能/能力面。**
- 本地权威：原始数据、DataAsset、Field、Record、Workspace、Space、Module、Mount、Metric、Proposal、Activity、本地派生分析结果。
- 云端能力面：身份、License/Entitlement、AI Token/Usage、LLM、长期 AI Profile、未来自主/定时 Agent Runtime。
- MVP 不做云同步；原始数据不默认上传云端；离线时本地与确定性分析可工作，云 LLM 能力提示「需要联网」。

## Alternatives

- A 纯 Local-first：数据全本地，AI 按需调云。
- B 纯 Cloud-first：桌面为薄客户端，数据全上云。
- C 变体：去掉本地权威（数据直接上云）。

## Rejected

- **A 纯 Local-first**：无法支撑团队/企业、跨设备、商业化额度计量、关 App 仍要运行的自主任务；后期被迫重造后端，迁移成本最高。
- **B 纯 Cloud-first**：数据默认上云，违背「用户拥有数据/可迁移/离线/不靠锁数据」，隐私代价高、MVP 复杂度高。
- **C 变体**：与 B 同理；丢失本地权威。

## Why

「数据权威(本地)」与「能力运行时(云)」解耦，是唯一同时满足数据归属与能力扩展的方案；也与「确定性计算本地执行、AI 在云理解」的产品原则一致。

## Consequences

- 需要明确本地/云边界与对象级 id/version/owner 同步缝（已入 schema）。
- 需要离线降级规则（本地可工作，云能力提示联网）。
- V1 自主/定时运行时放云端；Team/Enterprise 经 owner/scope 泛化扩展。
