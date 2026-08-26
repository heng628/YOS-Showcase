# 设计文档索引（YOS Showcase）

本目录为 YOS 的产品与架构设计文档（公开版本）。完整正文如下：

| 左侧导航 | 文档 |
|---|---|
| 产品 | `product/PRD.md` — 产品需求：核心闭环 / 数据资产原则 / AI 理解与确认 / MVP-V1 边界 |
| 架构 | `architecture/` — 领域模型（资产=用户级共享，Mount 复用）· Agent 架构（调用链与工具契约）· 权限模型（L0–L3 / A·B·C·D）· 状态模型（三轴正交）· 数据库 Schema · 技术选型 ADR |
| 决策 | `decisions/` — 11 篇 ADR（Hybrid 本地优先、Agent 不直连 DB、Proposal 一等对象、指标 DSL 等，均含"为何否决其它方案"） |
| 设计 | `design/design-system.md` — 设计 Token 与视觉规范 |

> 注：完整源码（monorepo：9 packages、201 tests、E2E 与探针）保存在作者的私有仓库；如需审阅请联系作者。
