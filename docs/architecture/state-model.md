# 状态模型（State Model）

- 状态：草案（待用户批准）
- 依据：`AGENTS.md` §30（8 条核心规范）；Step 9 状态设计（已确认）
- 铁律：**三轴正交，不把三个维度混成一个状态枚举。**

---

## 1. 三轴定义

| 轴 | 状态 | 说明 |
|---|---|---|
| **Lifecycle** | `Draft → Active → Archived → Deleted` | 受确认门槛的对象加 `Proposed` 作 Active 前态；`Empty` 是 Active 的派生，不是生命周期 |
| **Execution** | `Queued → Running → Paused → Completed / Failed / Cancelled` | 异步/持续过程 |
| **Health** | `Healthy / Warning / Error / Stale / Disconnected` | 数据类对象 |

- 每个对象只在需要的轴"开槽"；不要把 `Stale`(健康) 混成 `Failed`(执行)。

## 2. 逐对象状态机

### DataAsset
- Lifecycle：`Draft → Provisioning → PendingConfirmation → Active → Archived → Deleted(软删)`
- Health：`Healthy / Warning / Error / Stale / Disconnected`
- 触发：导入/分析/理解（Provisioning，执行中）→ 用户确认映射（PendingConfirmation）→ 生效（Active）；健康由同步/健康检查驱动。
- **Agent 自动**：只读/体检可自动；**不可自动**：映射确认、结构/原始数据变更、删除。

### DataStream
- Lifecycle：`Draft → Active → Paused → Disabled → Deleted(软删)`
- Execution：`Idle / Connecting / Syncing / Success / Failed`
- Health：`Healthy / AuthExpired / SchemaChanged / MappingInvalid / Disconnected`
- **Agent 自动**：同步/连接重试、状态标记；**必须用户**：重授权、重映射、停用/删除。

### Metric
- Lifecycle：`Proposed → Confirmed → Active → Deprecated → Deleted(软删)`；`Referenced` 为派生关系。
- 触发：AI 推荐 → Proposed；用户接受 → Confirmed → Active；删除=软删+引用预告（引用 Module 降级）。
- **Agent 自动**：推荐/提案；**必须确认**：创建/修改/删除。

### Module
- Lifecycle：`Creating → Active → Hidden → Archived → Deleted(软删)`
- **Agent 自动**：创建（B 类，可自动）、布局调整；**必须确认**：删除（且删 Module 绝不删数据/Metric/Mount）。

### Workspace
- Lifecycle：`Draft → Active → Archived → Deleted(软删)`；`Empty` = Active 派生（无内容，显示引导屏，不空白）。
- **Agent 自动**：创建（B）、改名/排序；**必须确认**：删除（软删，不删资产）。

### Monitoring
- Lifecycle：`Draft → PendingConfirmation → Active → Archived → Deleted(软删)`
- Execution/瞬时：`Triggered → Resolved`
- 语义独立态：`Muted`（静默但运行）≠ `Paused`（停止）≠ `Deleted`（删除）。
- "以后不要提醒" = **Mute + 负向学习信号**（非删除/非硬暂停）。

### Report
- Lifecycle：`Generating → Generated → Saved → Scheduled(V1) → Archived → Deleted`；`Failed / Interrupted`
- MVP：临时报告（会话内，不落库）；`Interrupted` 可恢复/重试；正式 Report 对象 V1。

### AgentTask
- Execution：`Queued → Planning → WaitingConfirmation → Running → Completed / PartiallyCompleted / Failed / Cancelled / Paused`
- 用户可见粗粒度（人话状态）；内部 Planning 微状态不暴露。

### Proposal
- Execution：`Proposed → Viewed → Accepted / Edited→Accepted / Rejected / Ignored / Expired → Executed → Done / Failed`；`Cancelled`
- **只由用户决策**（Accepted/Rejected/Ignored/Edited）；Executed 由 Application 在确认后驱动。

## 3. 触发权限（谁可以/谁必须/谁永不可）

| 状态转换 | Agent 自动 | 必须用户确认 | 永不自动 |
|---|---|---|---|
| 只读/分析/体检/推荐/提案 | ✅ | — | — |
| 创建低风险可逆对象（Module/Mount/Space/Workspace 等） | ✅(B) | 首次确认 | — |
| 创建知识资产（Metric/派生 Field） | — | ✅ C | — |
| 修改/删除持久对象 | — | ✅ C | — |
| 修改原始数据 / Field Type / Mapping | — | ✅ C | ✅(修改) |
| 删除原始数据 | — | — | ✅ **D** |
| 权限/账号/安全、外部副作用 | — | — | ✅ |
| 创建自主/定时行为 | — | ✅ 授权式 | — |
| 删除（一切对象） | — | ✅ C（软删+引用预告） | ✅(原始数据) |

## 4. 语义边界（不可混用）

- `Empty ≠ Draft ≠ Archived`
- `Muted ≠ Paused ≠ Deleted`
- `Stale(Health) ≠ Failed(Execution)`
- `PendingConfirmation` 属于流程/Proposal 侧，不是对象生命周期枚举（对象停在 `Draft/Proposed`，确认后入 `Active`）。

## 5. 用户可见性

- 面向用户只显示"人话"状态（状态词 + 阶段反馈）。
- 每个异步过程必须给可理解阶段反馈（如：正在读取数据 → 正在识别字段 → 正在计算指标 → 已发现 2 个问题），不用 "AI thinking"。
- 内部执行/健康微状态不直接暴露。
