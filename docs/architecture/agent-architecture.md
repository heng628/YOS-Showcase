# Agent 架构（Agent Architecture）

- 状态：**已批准**（批准范围：Slice Agent-1 最小单轮意图路由；propose/execute/AgentTask/自主运行时等其余部分仍待后续实施时再逐项批准）
- 依据：`AGENTS.md` §07/§19/§30/§31；`permission-model.md`、`domain-model.md`
- 核心定位：**Agent = 自主分析 + 受控执行**。
  - 自主性只存在于：发现、查询、分析、推理、规划、提出建议。
  - 不可自主突破：权限边界、数据写入边界、原始数据保护、外部访问权限、用户确认机制。

---

## 0. 调用链（强制）

```
User Intent
→ Agent(规划/发现/选择工具)
→ Tool Layer(白名单/typed)
→ Application Service(权限+确认闸门)
→ Domain(业务规则/状态机/确定性公式)
→ Repository → Data(SQLite 目录 / DuckDB 记录)
外部访问：
Agent → Connector/Sync Service → 用户授权 scope → License/Entitlement → External API
```

**禁止**：`Agent → DB`、`Agent → Repository`、`Agent → 任意文件系统写入口`、`Agent → 绕过 Application`、`Agent → 绕过确认机制`。

---

## 17 个关键问题

### 1. Agent 如何发现相关 Data Asset？
通过 Tool Layer 暴露的只读工具 `list/describe` 能力：按用户意图语义匹配（名称/字段语义/最近使用/健康度/AI 画像偏好），返回候选资产。**只读、无需确认**。

### 2. Agent 如何发现 Field / Metric？
同一只读工具：`describe_asset(assetId)` → 字段清单/类型/语义角色；`search_metrics(intent)` → 用户级指标库（按语义与使用频次匹配）。同样只读。

### 3. Agent 如何选择工具？
工具清单由 **Tool Registry（Application 层）** 静态提供（白名单）；Agent 在给定工具的 schema 描述下自主选择——「怎么分析」自主，「有哪些工具」不自主。

### 4. Tool 如何定义？
```
Tool = {
  name,                    // 工具名
  purpose,                 // 用途说明
  inputSchema,             // typed 输入 schema（严格限参）
  outputSchema,            // typed 输出 schema（结构化返回）
  riskLevel,               // low / med / high
  access,                  // read / write
  requiresConfirmation,    // boolean
  allowedActor,            // user / ai
  confirmationPolicy,      // A / B / C / D（见 permission-model.md）
}
```
定义属于 Application 层；Agent 只消费（白名单）。

### 5. Tool 是否 typed？
**是。** 每个工具带 **typed inputSchema + typed 返回**（结构化），LLM 只能按 schema 填参；返回为可校验的结构化结果。

### 6. Tool 如何限制参数？
- 输入 schema 严格限定字段/类型/枚举/范围；
- Application 在**执行前再次校验**参数（越界/越权/非法引用一律拒绝）；
- 数据范围限定在当前用户 owner 作用域内。

### 7. Tool 如何区分 read / analyze / propose / execute？
按 `kind` 与 `confirmationPolicy` 分四类：

| kind | 含义 | 确认 |
|---|---|---|
| `read` | 查询/列表/描述（L0） | A 直接 |
| `analyze` | 计算/分析/推理/临时洞察（L1，确定性本地计算） | A 直接 |
| `propose` | 生成写操作 Proposal（不落业务写） | 产生 Proposal |
| `execute` | 由**已确认 Proposal** 驱动的执行（Application 内部调用，**不对 Agent 直接开放**） | 仅经确认后 |

> 关键：**Agent 永远接触不到 `execute` 类工具**——execute 由 Application 在用户确认后自行调用。Agent 的写能力止步于 `propose`。

### 8. Agent 如何产生 Proposal？
Agent 完成分析后，若要持久写入/结构变化/外部副作用/自主行为 → 通过 `propose_*` 工具生成一个 **Proposal（一等对象）**：动作、原因、before/after、影响、可撤销性、风险等级、置信度、目标对象。

### 9. 谁负责 Confirmation？
**用户**（Confirmation Card 呈现 Proposal 的决策：确认/修改/拒绝/忽略）。系统只负责按 `confirmationPolicy` 决定哪些必须等确认。

### 10. 谁负责真正执行？
**Application 层**（在 Proposal 状态 = Accepted 后执行；高危要求再核对影响与可撤销）。Agent 无执行权。

### 11. 谁负责 Activity Log？
**Application 层统一写**（所有执行结果——含失败与未确认的只读重大动作——一律落日志）。Agent 不得直接写日志。

### 12. Agent 如何避免直接访问 DB？
工程上三层强制：
1. **包边界**：`packages/agent` 不依赖 `packages/database`（pnpm + import 规则/CI 校验）。
2. **运行时边界**：Agent 进程上下文只持有 Tool 客户端，不持有任何 Repository/连接句柄。
3. **工具白名单**：只暴露 read/analyze/propose 工具；execute/仓库接口不出现在工具表。

### 13. Agent 如何处理低置信度？
- 建议/洞察带置信度标注；低置信度时**明示"推测，请确认"**；
- 涉及写操作的提案：置信度低 → 降级为**更强确认**（甚至默认不作为自动推荐，只作候选）；
- 绝不因低置信度而扩大权限或跳过确认。

### 14. Agent 如何处理用户拒绝？
- 记录 `Rejected` + 作为**负向学习信号**（AI 画像）；
- 允许继续分析，但**不得重复执行/反复弹出同一被拒绝动作**（§31 第 8 条）；
- 只有用户主动改变要求时，才可重新提议。

### 15. Agent 如何处理工具失败？
- 失败结果结构化返回：`{ok:false, reason, retryable}`；
- 可重试的 → 有限自动重试（读类）后仍失败则如实报告；
- 不可重试/写类 → 中止该分支，报告失败原因，**不私自换路径绕过**；
- 全部记入 Activity Log。

### 16. Agent 如何处理权限不足？
- 只报告「需要更高权限/需要用户操作」（如重授权、重映射、需用户确认的高风险），
- **绝不尝试自行提升权限、绝不绕过确认**，给出用户可执行的具体步骤。

### 17. Agent 如何处理外部 Connector？
- 只经 **Connector/Sync Service**（V1 才有真实实现；MVP 仅文件/手动）。
- 外部访问必须：用户授权 scope → License/Entitlement 检查 → 才可调用外部 API。
- Agent 不接触凭据、不自行决定访问范围。

---

## Agent 会话流程（完整）

```
User Intent
→ Agent 规划(目标分解)
→ 发现数据(只读工具) → 选择工具(白名单) → 分析/推理(确定性引擎本地算, LLM 只解释)
→ 形成 洞察(直接呈现) 或 Proposal(一等对象)
→ [写操作] Confirmation Card → 用户 确认/修改/拒绝/忽略
→ Application 执行(校验+事务+软删/版本)
→ 验证结果 → Activity Log
```

## Agent Task 形态

- **MVP：会话内任务**（Queued→Planning→WaitingConfirmation→Running→Completed/Failed），随会话结束。
- **V1：定时/自主任务**（云端运行时，关掉 App 仍运行）；`schedule` 只预留字段，MVP 不实现。

## 核心原则（重述）

> Agent 可以自主决定「怎么分析」。Agent 不能自主决定「是否拥有更高权限」。
