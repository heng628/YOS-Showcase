# 领域模型（Domain Model）

- 状态：草案（待用户批准）
- 依据：`AGENTS.md` §02/§03/§19/§20/§30/§31；Step 1–9 已确认决策
- 总原则：
  - **Data Asset 是用户级顶层共享资产，不属于任何 Workspace。**
  - **Workspace 通过 Mount 使用 Data Asset；Module 通过 Mount + Metric 使用数据。**
  - **Metric 是独立用户级知识资产。**
  - 所有对象三轴状态：Lifecycle × Execution × Health（详见 `state-model.md`）。
  - MVP owner = User（单用户），所有对象带 `ownerId/scopeId` 缝，为 Team/Enterprise 预留。

---

## 对象总表

| 对象 | 职责 | owner/scope | MVP |
|---|---|---|---|
| User | 产品用户（MVP 单用户） | 自身 | ✅ |
| Space | 工作场景分组容器（非数据隔离） | User | ✅ |
| Workspace | 具体工作环境（使用/组织数据） | User（挂在 Space 下） | ✅ |
| DataAsset | 用户级数据资产（顶层共享） | **User（绝不属 Workspace）** | ✅ |
| Field | 数据资产的原子/行级派生字段 | 所属 DataAsset（owner 继承 User） | ✅ |
| Record | 原始数据行（追加/版本，权威） | 所属 DataAsset（owner 继承 User） | ✅ |
| Connection | 访问通道（文件/账号/手工入口） | User | ✅（仅文件/手工） |
| DataSource | 数据品类定义（可复用） | User | ✅（仅文件/手工） |
| DataStream | 接入/同步实例（Connection×DataSource，N:M 连资产） | User | ✅（一次性导入） |
| Mount | Workspace 对某 DataAsset 的使用视图 | Workspace | ✅ |
| Metric | 用户级分析知识资产（引用≥1 字段） | **User（不属 DataAsset）** | ✅ |
| Module | Workspace 内观察/轻量工作单元 | Workspace | ✅ |
| Monitoring | 监测规则/持续观察行为 | User（工作台作用域视图） | ✅ 基础 |
| Report | 分析输出 | User（工作台作用域视图） | ⚠️ MVP 仅临时/简单导出，正式对象 V1 |
| Proposal | Agent 权限的原子单位（一等对象） | User | ✅ |
| Confirmation | **Proposal 的确认交互/决策（UI 表现，非独立对象）** | — | ✅ |
| ActivityLog | Agent/用户操作流水（可读、可撤销追溯） | User | ✅ |
| AgentTask | Agent 任务（MVP 会话内；V1 定时/自主） | User | ✅ 基础 |

---

## 1. User

- 职责：产品用户；MVP 单用户、本地权威。
- owner/scope：自身。
- 核心字段：`id(UUID)`、`name`、`locale`、`simpleModeDefault`、`createdAt/updatedAt`。
- 生命周期：Active（无删除语义；归档/删除不做）。
- 关系：`1─N Space / Workspace / DataAsset / Metric / Connection / Monitoring / Report / Proposal / ActivityLog / AgentTask`。
- 创建/修改/删除：系统（首次启动）创建；用户改资料；不可删。
- Agent 可操作：只读；不得改权限/账号（安全红线）。

## 2. Space

- 职责：工作场景分组（工作/学习/个人…），**只装 Workspace，不拥有、不隔离数据**。
- owner/scope：User。
- 核心字段：`id(UUID)`、`ownerId`、`name`、`icon`、`sort`、`themeRef?`、`lifecycle`、`createdAt/updatedAt/deletedAt`。
- 生命周期：`Draft → Active → Archived → Deleted(软删)`。
- 关系：`1─N Workspace`。
- 创建：用户/AI(提案)；修改：用户/AI(提案)；删除：用户确认（软删，级联检查 Workspace）。
- Agent 可操作：创建/改/删均走 Proposal→确认；**删除是软删，绝不连带删数据资产**。

## 3. Workspace

- 职责：具体工作环境；承载数据(经 Mount)、模块、监测/报告作用域视图、设置。
- owner/scope：User（`spaceId` 定位）。
- 核心字段：`id(UUID)`、`ownerId`、`spaceId`、`name`、`mode(simple/pro)`、`sort`、`lifecycle`、`createdAt/updatedAt/deletedAt`。
- 生命周期：`Draft → Active → Archived → Deleted(软删)`；`Empty` 是 Active 的派生（无内容），非生命周期。
- 关系：`Space 1─N Workspace`；`Workspace 1─N Mount`；`Workspace 1─N Module`。
- 创建：用户/AI(提案)；修改：用户/AI(提案)；删除：用户确认（软删；其 Mount/Module 随工作台归档，**数据资产与指标不受影响**）。
- Agent 可操作：创建/改/删 → Proposal→确认；**删除工作台绝不删 DataAsset/Metric**。

## 4. DataAsset

- 职责：用户级数据资产（订单/广告/客户…），**顶层共享、可被多个 Workspace 经 Mount 使用、原始数据只存一份**。
- owner/scope：**User（绝不属 Workspace）**。
- 核心字段：`id(UUID)`、`ownerId`、`name`、`kind(singleSource)`、`health`、`lifecycle`、`dataVersion`、`rowCount`、`sourceSummary`、`createdAt/updatedAt/deletedAt`。
- 生命周期：`Draft → Provisioning → PendingConfirmation → Active → Archived → Deleted(软删)`；健康轴：`Healthy/Warning/Error/Stale/Disconnected`。
- 关系：`User 1─N DataAsset`；`DataAsset 1─N Field`；`DataAsset 1─N Record`；`DataAsset ←N:M→ DataStream`；`Mount N─1 DataAsset`。
- 创建：用户（导入/手动/建结构）或 AI(提案，导入后确认)；修改：仅结构/映射变更时确认；删除：用户确认（软删 + 引用预告：告知被哪些 Workspace/Metric 使用）。
- Agent 可操作：只读/分析/体检/建议；**原始数据/结构/映射的修改删除永不自动（C/D）**。

## 5. Field

- 职责：数据资产的字段（原子类型 + 行级派生字段）。**原子值类型**（Text/Number/Currency/Percentage/Date/DateTime/Boolean/SingleSelect/MultiSelect/URL/File/ID）；**Formula/Relation 不是字段类型**（Formula→Metric 定义；Relation→独立对象，V1）。
- owner/scope：所属 DataAsset（owner 继承 User）。
- 核心字段：`id(UUID)`、`dataAssetId`、`name`、`displayName`、`type`、`unit?`、`semanticRole?`、`mapping?`、`lifecycle`、`createdAt/updatedAt/deletedAt`。
- 生命周期：`Draft → Active → Deprecated → Deleted(软删)`。
- 关系：`DataAsset 1─N Field`；`Metric N─M Field`（经 MetricInput）。
- 创建：导入推断(AI 提议→用户确认)/用户手动；修改：`Field 类型/映射 = C（每次确认，高危）`；删除：用户确认（引用检查）。
- Agent 可操作：发现/提议类型与语义；**修改 Field 类型/映射永不自动（C）**。

## 6. Record

- 职责：原始数据行，**本地权威、追加/版本化**（不覆盖），存于 DuckDB（每资产一个记录库）。
- owner/scope：所属 DataAsset（owner 继承 User）。
- 核心字段：资产字段列 + 系统列 `_recordId`、`_version`、`_batchId(导入/同步批)`、`_ingestedAt`、`_isDeleted(软删)`。
- 生命周期：追加为主；行级软删；版本=新版本行（保留旧版本，可回滚）。
- 关系：`DataAsset 1─N Record`。
- 创建：导入/录入（经 DataStream/手动）；修改：**C（每次确认）**；删除：**D（Agent 永不执行，仅用户）**。
- Agent 可操作：只读/聚合/分析；**永不自动写/删原始行**。

## 7. Connection

- 职责：访问通道（"以何身份、从哪获取"）。MVP 仅 `file` / `manual`；V1 加 API/OAuth。
- owner/scope：User。
- 核心字段：`id(UUID)`、`ownerId`、`kind(file/manual/api/oauth)`、`name`、`status`、`credentialRef?`、`lifecycle`、`createdAt/updatedAt/deletedAt`。
- 生命周期：`Draft → Active → Disconnected → Deleted(软删)`。
- 关系：`User 1─N Connection`；`Connection 1─N DataStream`。
- 创建：用户；修改：用户；删除：**用户确认；删除 Connection 不删已沉淀数据资产**（历史保留）。
- Agent 可操作：只读状态/提醒重连；不得管理凭据/授权（安全红线）。

## 8. DataSource

- 职责：数据品类定义（"能提供什么数据"），可复用（内容数据/广告数据/某 Excel 的销售表）。
- owner/scope：User。
- 核心字段：`id(UUID)`、`ownerId`、`connectionId`、`name`、`kind`、`schemaDef?`、`lifecycle`、`createdAt/updatedAt/deletedAt`。
- 生命周期：`Draft → Active → Archived → Deleted(软删)`。
- 关系：`Connection 1─N DataSource`；`DataSource 1─N DataStream`。
- 创建：用户（或 AI 建议→确认）；修改：用户；删除：用户确认（引用检查）。
- Agent 可操作：发现/建议可用来源；不自主修改定义。

## 9. DataStream

- 职责：接入/同步实例（Connection × DataSource），承载状态/频率/映射/健康；**N:M 连接资产**。
- owner/scope：User。
- 核心字段：`id(UUID)`、`ownerId`、`connectionId`、`dataSourceId`、`dataAssetId`、`mode(oneShot/sync)`、`mapping`、`lifecycle`、`execution`、`health`、`lastSyncAt`、`createdAt/updatedAt/deletedAt`。
- 生命周期：`Draft → Active → Paused → Disabled → Deleted(软删)`；执行：`Idle/Connecting/Syncing/Success/Failed`；健康：`Healthy/AuthExpired/SchemaChanged/MappingInvalid/Disconnected`。
- 关系：`Connection×DataSource → DataStream`；`DataStream N─1 DataAsset`（多流供一资产）。
- 创建：用户（或 AI 提案→确认）；修改：映射变更=**C**；删除：用户确认（引用检查）。
- Agent 可操作：同步重试/状态标记可自动；**重映射/重授权必须用户确认**。

## 10. Mount

- 职责：Workspace 对某 DataAsset 的**使用视图**（"这个工作台用它"）；不含指标定义、不含展示配置。
- owner/scope：Workspace（owner 继承 User）。
- 核心字段：`id(UUID)`、`workspaceId`、`dataAssetId`、`alias?`、`scopeFilter?`、`lifecycle`、`createdAt/updatedAt/deletedAt`。
- 生命周期：`Active → Archived → Deleted(软删)`。
- 关系：`Workspace 1─N Mount N─1 DataAsset`；`Module N─M Mount`。
- 创建：用户/AI(提案)；删除：用户确认（引用检查：被 Module 使用时提示）。
- Agent 可操作：创建/删除走 Proposal→确认；**删 Mount 不删 DataAsset**。

## 11. Metric

- 职责：**用户级分析知识资产**。定义=对≥1 个字段的确定性聚合/业务公式；`dataScope = single/cross`（属性，非类型）；`origin = system/user/ai`；可被多工作台多模块复用，定义唯一。
- owner/scope：**User（不属任何 DataAsset）**。
- 核心字段：`id(UUID)`、`ownerId`、`name`、`definition(formula)`、`unit`、`defaultFormat`、`origin`、`dataScope`、`lifecycle`、`version`、`createdAt/updatedAt/deletedAt`。
- 生命周期：`Proposed → Confirmed → Active → Deprecated → Deleted(软删)`；`Referenced` 是派生关系非状态。
- 关系：`User 1─N Metric`；`Metric N─M Field`（经 MetricInput）；`Module N─M Metric`（经 ModuleMetricBinding）。
- 创建：用户手动 / AI Proposal→确认；修改：`C`（影响引用，预览影响）；删除：软删 + 引用预告（被引用 Module 降级为失效占位）。
- Agent 可操作：推荐/提案（L1）；**创建/改/删必须 Proposal→确认**。

## 12. Module

- 职责：Workspace 内的观察/轻量工作单元。= 数据作用域(Mount+Metric) + 展示配置 + 可选动作。**不拥有数据**。MVP 3 种展示：KPI / Chart / Table（Insight=Context 行+系统层；AI=横切能力，均非模块类型）。
- owner/scope：Workspace（owner 继承 User）。
- 核心字段：`id(UUID)`、`workspaceId`、`name`、`presentation(kpi/chart/table)`、`layout`、`lifecycle`、`createdAt/updatedAt/deletedAt`。
- 生命周期：`Creating → Active → Hidden → Archived → Deleted(软删)`。
- 关系：`Workspace 1─N Module`；`Module N─M Metric`（ModuleMetricBinding 含展示配置）；`Module N─M Mount`（ModuleMountBinding）。
- 创建：用户/AI(轻确认，B 类可自动)；修改：用户/AI(轻确认)；删除：用户确认。
- Agent 可操作：创建/改布局/复制 → 低风险（B/C）；**删 Module 绝不删数据/Metric/Mount**。

## 13. Monitoring

- 职责：监测规则/持续观察行为（`{指标, 范围, 条件, 频率}`）。对象全局（挂指标），工作台提供作用域视图。
- owner/scope：User（可关联 workspaceId）。
- 核心字段：`id(UUID)`、`ownerId`、`workspaceId?`、`name`、`metricId`、`rule(条件/频率/敏感度)`、`lifecycle`、`execution`、`muted`、`createdAt/updatedAt/deletedAt`。
- 生命周期：`Draft → PendingConfirmation → Active → Archived → Deleted(软删)`；触发态 `Triggered/Resolved`；`Muted/Paused` 语义独立。
- 创建：用户/AI(提案→确认，授权式)；修改：用户（含 Mute/敏感度）；删除：用户确认。
- Agent 可操作：分析异常=直接；**创建/修改规则必须确认**；"以后不要提醒" = Mute + 负向学习信号（非删除/非硬暂停）。

## 14. Report

- 职责：分析输出。**MVP 只做"临时报告/简单导出"，正式 Report 对象与报告中心在 V1**（依据 AGENTS.md §21/§22）。
- owner/scope：User（可关联 workspaceId）。
- 核心字段（V1 正式）：`id`、`ownerId`、`workspaceId?`、`title`、`content/layout`、`lifecycle`、`execution`、`schedule?`、`createdAt/updatedAt/deletedAt`。
- 生命周期（V1）：`Generating → Generated → Saved → Scheduled → Archived → Deleted`；`Failed/Interrupted`。
- MVP 呈现：临时报告=会话内生成物（不落库，可导出）；落库的 Report 对象 V1 再上。

## 15. Proposal

- 职责：**Agent 权限体系的原子单位（一等对象）**。承载 动作/原因/改前改后/影响/可撤销/置信度/对象/提案者/用户决策；持久、可过期、可审计；是 Activity 与撤销的依据。
- owner/scope：User。
- 核心字段：`id(UUID)`、`ownerId`、`agentTaskId?`、`action`、`reason`、`before/after(diff)`、`impact`、`reversible`、`riskLevel(low/med/high)`、`confidence?`、`target(objectRef)`、`proposer(user/ai)`、`status(Proposed/Viewed/Accepted/Rejected/Ignored/Edited/Expired/Executed/Failed/Cancelled)`、`decidedAt`、`createdAt/updatedAt`。
- 生命周期（执行轴）：`Proposed → Viewed → Accepted/Edited→Accepted/Rejected/Ignored/Expired → Executed → Done/Failed`；`Cancelled`。
- 关系：`User 1─N Proposal`；`Proposal 1─N ActivityLog`；`AgentTask 1─N Proposal`。
- 创建：AI（或用户触发的写动作包装）；决策：**用户**（确认/修改/拒绝/忽略）。
- Agent 可操作：只能创建 Proposal；**执行权在用户确认之后，由 Application 执行**。

## 16. Confirmation

- 职责：**Proposal 的确认交互 + 用户决策，是 Proposal 的 UI 表现与决策动作，不设独立实体**（依据 §30 第 2 条）。
- 落点：Proposal 上的 `status/decidedAt` + Confirmation Card（UI 渲染：默认显示 做什么/为什么/改什么 + 确认/修改/拒绝/忽略；展开 影响/可撤销/涉及数据/工作台与模块/置信度/详细 diff；高风险强制展开影响与可撤销）。

## 17. ActivityLog

- 职责：Agent/用户操作流水（可读，非开发者日志）。`时间 / 做了什么 / 为什么 / 读了什么 / 提出了什么 / 是否确认 / 结果 / 是否可撤销`。
- owner/scope：User。
- 核心字段：`id(UUID)`、`ownerId`、`actor(user/ai)`、`action`、`summary`、`targetRef?`、`proposalId?`、`result`、`reversible`、`createdAt`。
- 生命周期：追加（append-only，可归档）。
- 创建：**所有执行结果必须写入**（§31 第 9 条），由 Application 层统一写；不可由 Agent 直接写。
- Agent 可操作：无直接写权限；只经 Application 落日志。

## 18. AgentTask

- 职责：Agent 任务。MVP：会话内任务（发现/分析/规划/提案）；V1：定时/自主（云端运行时，预留 `schedule` 缝）。
- owner/scope：User。
- 核心字段：`id(UUID)`、`ownerId`、`name`、`goal`、`plan`、`lifecycle`、`execution`、`progress`、`schedule?`(V1)、`createdAt/updatedAt/deletedAt`。
- 生命周期：`Queued → Planning → WaitingConfirmation → Running → Completed/PartiallyCompleted/Failed/Cancelled/Paused`。
- 关系：`User 1─N AgentTask`；`AgentTask 1─N Proposal`。
- 创建：用户意图触发；执行：Application/Agent（受权限模型约束）。
- Agent 可操作：只读/分析阶段自主；**写阶段一律经 Proposal→确认**。

---

## 关键关系总图（概念级）

```
User 1─N Space 1─N Workspace
User 1─N DataAsset 1─N Field；DataAsset 1─N Record
User 1─N Metric；Metric N─M Field (MetricInput)
User 1─N Connection 1─N DataSource 1─N DataStream N─1 DataAsset
Workspace 1─N Mount N─1 DataAsset
Workspace 1─N Module；Module N─M Metric (ModuleMetricBinding)；Module N─M Mount (ModuleMountBinding)
User 1─N Proposal；User 1─N ActivityLog；User 1─N AgentTask 1─N Proposal
Monitoring / Report 挂 User（可选关联 workspaceId，作用域视图）
```

## 创建/修改/删除权限总表

| 对象 | 创建 | 修改 | 删除 | Agent 可操作 |
|---|---|---|---|---|
| User | 系统 | 用户 | 无 | 只读 |
| Space | 用户/AI(提案) | 用户/AI(提案) | 用户确认(软删) | 提案→确认 |
| Workspace | 用户/AI(提案) | 用户/AI(提案) | 用户确认(软删,不删资产) | 提案→确认 |
| DataAsset | 用户/AI(提案+导入确认) | 结构/映射=确认 | 用户确认(软删+引用预告) | 只读/建议；C/D 护栏 |
| Field | 导入推断(确认)/用户 | **类型/映射=C** | 用户确认(引用检查) | 提议；改类型=永不自动 |
| Record | 导入/录入 | C | **D(仅用户)** | 只读/聚合 |
| Connection | 用户 | 用户 | 用户确认(不删资产) | 只读/提醒 |
| DataSource | 用户/AI(提案) | 用户 | 用户确认 | 提案→确认 |
| DataStream | 用户/AI(提案) | 映射=C | 用户确认 | 重试/标记自动；重映射=确认 |
| Mount | 用户/AI(提案) | 用户 | 用户确认 | 提案→确认 |
| Metric | 用户/AI(提案) | C | 软删+引用预告 | 提案→确认 |
| Module | 用户/AI(轻确认) | 用户/AI(轻确认) | 用户确认 | B/C 低风险 |
| Monitoring | 用户/AI(授权式确认) | 用户 | 用户确认 | 提案→确认 |
| Report | 临时=会话内 | — | — | 只读/生成(临时) |
| Proposal | AI/写动作包装 | 用户(决策) | 过期/取消 | 只能创建 |
| ActivityLog | Application 统一写 | 无 | 归档 | 不可直写 |
| AgentTask | 用户意图 | Application | 用户取消 | 执行受权限模型 |
