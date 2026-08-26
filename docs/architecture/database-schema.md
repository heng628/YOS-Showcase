# 数据库 Schema 草案（Database Schema Draft）

- 状态：草案（待用户批准）；**只定义表与实体关系，不实现数据库**
- 依据：`domain-model.md`、`technical-stack-adr.md`、`AGENTS.md` §20/§30
- 存储分工（本地权威，MVP 无云同步）：
  - **SQLite = 系统目录/配置**（对象、实体、关系、Proposal、Activity、任务）
  - **DuckDB = 原始记录（每 DataAsset 一个记录库）+ 确定性分析引擎的数据源**
  - **派生结果 = 独立可重算存储**（指标结果缓存等，可由原始数据重算，绝不写回原始数据）

---

## 0. 全表通用约定（同步缝 / 版本 / 软删）

- **主键**：一律 `id TEXT PRIMARY KEY`（UUID，跨设备稳定——未来云同步的关键缝）。
- **归属**：所有对象带 `owner_id`（MVP 恒为唯一 User 的 id）；`scope_id` 预留（MVP = owner_id，未来 Team/Org 使用）。
- **版本**：`version INTEGER NOT NULL DEFAULT 1`（乐观更新；每次修改 +1——同步/回滚缝）。
- **时间**：`created_at`、`updated_at`（UTC ISO 字符串）。
- **软删除**：`deleted_at TEXT NULL`（NULL=未删）+ 三轴状态列；**一律软删，不硬删**。
- **三轴状态**：`lifecycle`、`execution`、`health` 三个 TEXT 列，只在需要的轴上有值（见 `state-model.md`）。
- **MVP 所有数据本地权威**；本 Schema 只为未来同步预留稳定 id/version/owner，**不实现同步**。

---

## 1. 目录/配置（SQLite）

### user
| 列 | 说明 |
|---|---|
| id (PK, UUID) | 用户 |
| name, locale | |
| simple_mode_default | Simple/Pro 默认 |
| version, created_at, updated_at | 通用列 |

### space
| 列 | 说明 |
|---|---|
| id (PK, UUID) | |
| owner_id, scope_id | User |
| name, icon, sort | 可增删改名排序 |
| theme_ref | 主题引用（可空，V1） |
| lifecycle, version, created_at, updated_at, deleted_at | |

### workspace
| 列 | 说明 |
|---|---|
| id (PK, UUID) | |
| owner_id, scope_id, space_id (FK→space) | User / Space |
| name, mode(simple/pro), sort | |
| lifecycle, version, created_at, updated_at, deleted_at | |

### data_asset
| 列 | 说明 |
|---|---|
| id (PK, UUID) | |
| owner_id, scope_id | **User（绝无 workspace_id）** |
| name, kind(single_source), row_count, data_version | 元数据 |
| lifecycle, health, version, created_at, updated_at, deleted_at | 三轴中的 Lifecycle+Health |

> ⚠️ `data_asset` **没有 workspace_id**：DataAsset 是用户级顶层共享资产；Workspace 只能经 `mount` 使用它。

### field
| 列 | 说明 |
|---|---|
| id (PK, UUID) | |
| owner_id, scope_id, data_asset_id (FK) | 所属资产 |
| name, display_name, type(原子枚举), unit, semantic_role, mapping | 类型=原子类型（Text/Number/Currency/Percentage/Date/DateTime/Boolean/SingleSelect/MultiSelect/URL/File/ID） |
| lifecycle, version, created_at, updated_at, deleted_at | |

### metric（用户级分析资产）
| 列 | 说明 |
|---|---|
| id (PK, UUID) | |
| owner_id, scope_id | **User（绝无 data_asset_id）** |
| name, definition(公式 JSON), unit, default_format | 确定性定义 |
| origin(system/user/ai), data_scope(single/cross) | 属性 |
| lifecycle, version, created_at, updated_at, deleted_at | |

### metric_input（Metric N─M Field）
| 列 | 说明 |
|---|---|
| id (PK, UUID) | |
| metric_id (FK), data_asset_id (FK), field_id (FK), role | 指标引用字段（可跨资产） |
| version, created_at, updated_at | |

### mount（Workspace ↔ DataAsset）
| 列 | 说明 |
|---|---|
| id (PK, UUID) | |
| workspace_id (FK), data_asset_id (FK) | Workspace 1─N Mount N─1 DataAsset |
| alias, scope_filter | 使用视图配置 |
| lifecycle, version, created_at, updated_at, deleted_at | |

### module
| 列 | 说明 |
|---|---|
| id (PK, UUID) | |
| workspace_id (FK), owner_id | 所属工作台 |
| name, presentation(kpi/chart/table), layout | MVP 3 种展示 |
| lifecycle, version, created_at, updated_at, deleted_at | |

### module_metric_binding（Module N─M Metric + 展示配置）
| 列 | 说明 |
|---|---|
| id (PK, UUID) | |
| module_id (FK), metric_id (FK) | |
| display_config (JSON：时间窗/过滤/对比/图表) | 展示配置属于 Module，不属于 Metric |
| lifecycle, version, created_at, updated_at | |

### module_mount_binding（Module N─M Mount）
| 列 | 说明 |
|---|---|
| id (PK, UUID) | |
| module_id (FK), mount_id (FK) | 模块数据作用域 |
| version, created_at, updated_at | |

### monitoring
| 列 | 说明 |
|---|---|
| id (PK, UUID) | |
| owner_id, workspace_id (可空) | 全局对象 + 工作台作用域视图 |
| name, metric_id (FK), rule(JSON: 条件/频率/敏感度), muted | |
| lifecycle, execution, health, version, created_at, updated_at, deleted_at | |

### report（V1 正式对象；MVP 只做会话内临时报告）
| 列 | 说明 |
|---|---|
| id (PK, UUID) | |
| owner_id, workspace_id (可空) | |
| title, content(JSON), schedule(可空, V1) | |
| lifecycle, execution, version, created_at, updated_at, deleted_at | |

### proposal（一等对象）
| 列 | 说明 |
|---|---|
| id (PK, UUID) | |
| owner_id, agent_task_id (可空) | |
| action, reason | 做什么/为什么 |
| before_state, after_state (JSON/diff) | 改什么 |
| impact (JSON), reversible, risk_level(low/med/high), confidence | 影响/可撤销/风险/置信度 |
| target_ref (objectRef), proposer(user/ai) | 对象/提案者 |
| status (Proposed/Viewed/Accepted/Rejected/Ignored/Edited/Expired/Executed/Failed/Cancelled), decided_at | 决策 |
| version, created_at, updated_at | |

### activity_log（append-only）
| 列 | 说明 |
|---|---|
| id (PK, UUID) | |
| owner_id, actor(user/ai) | |
| action, summary, target_ref, proposal_id (可空) | |
| result, reversible, created_at | |

### agent_task
| 列 | 说明 |
|---|---|
| id (PK, UUID) | |
| owner_id, name, goal, plan(JSON) | |
| lifecycle, execution, progress, schedule(可空, V1) | |
| version, created_at, updated_at, deleted_at | |

### connection
| 列 | 说明 |
|---|---|
| id (PK, UUID) | |
| owner_id, kind(file/manual/api/oauth), name, status, credential_ref(可空) | MVP 仅 file/manual |
| lifecycle, version, created_at, updated_at, deleted_at | |

### data_source
| 列 | 说明 |
|---|---|
| id (PK, UUID) | |
| owner_id, connection_id (FK), name, kind, schema_def(JSON, 可空) | |
| lifecycle, version, created_at, updated_at, deleted_at | |

### data_stream（Connection×DataSource → DataAsset）
| 列 | 说明 |
|---|---|
| id (PK, UUID) | |
| owner_id, connection_id (FK), data_source_id (FK), data_asset_id (FK) | |
| mode(one_shot/sync), mapping(JSON), last_sync_at | |
| lifecycle, execution, health, version, created_at, updated_at, deleted_at | |

---

## 2. 原始记录（DuckDB，每 DataAsset 一个记录库）

- 表名：`asset_<dataAssetId>`（按资产的 Field 定义动态建列，列类型 = Field 原子类型）。
- 系统列（所有记录表统一）：
  - `_record_id TEXT`（行级稳定 id）
  - `_version INTEGER`（追加/版本；**不覆盖**）
  - `_batch_id TEXT`（导入/同步批）
  - `_ingested_at TEXT`
  - `_is_deleted INTEGER`（行级软删）
- 语义：**追加 + 版本化**；"修正/回填"= 新版本行，旧版本保留（可回滚）。
- **Agent 永不直连**；读写一律经 `RecordStore`（`packages/database` 内）→ Application。

## 3. 派生结果（可重算，独立于原始数据）

- 位置：本地（SQLite `metric_result_cache` 或 DuckDB 聚合视图）。
- 内容：指标计算结果/聚合缓存/AI 洞察引用。
- 原则：**可由原始数据重算；绝不写回原始记录表**；可随时清除重建。

---

## 4. 关系清单（FK 方向）

```
user 1─N space / workspace / data_asset / metric / connection / monitoring / proposal / activity_log / agent_task
space 1─N workspace
data_asset 1─N field；data_asset 1─N record(DuckDB)
metric N─M field (metric_input)
connection 1─N data_source 1─N data_stream N─1 data_asset
workspace 1─N mount N─1 data_asset
workspace 1─N module
module N─M metric (module_metric_binding)；module N─M mount (module_mount_binding)
agent_task 1─N proposal；proposal 1─N activity_log
monitoring / report：挂 owner（可选 workspace_id）
```

## 5. 同步缝清单（仅预留，不实现）

- 全表：稳定 `id(UUID)` + `version` + `created_at/updated_at` + `deleted_at`(软删) + `owner_id/scope_id`。
- 记录：`_record_id` + `_version` + `_batch_id`（可增量同步）。
- **MVP 不实现云同步**；本清单只保证将来加同步协议时**不需要改表结构**。
