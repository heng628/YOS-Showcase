# 010 — 升级前备份、迁移安全与发布态数据路径

- Date: 2026-08-24
- Status: **Accepted**
- 关联 Source of Truth: `database-schema.md`（迁移/存储分工）、`technical-stack-adr.md`、AGENTS.md §26

## Context

软件升级时新迁移会修改 `userData/yos.db`。当前迁移是内联 SQL + `__migrations` 表 + 单事务原子，但没有「迁移前备份」，也没有「迁移/初始化失败的用户反馈」；更严重的是，SQLite 打开失败时工厂会**静默降级为内存库**——发布场景等于用户数据「看起来消失」。必须在不引入复杂 rollback/备份框架的前提下补齐最低数据安全。

## Decision

1. **固定数据路径**：`productName = "YOS"`（写入 `apps/desktop/package.json` 与 electron-builder 配置）。userData 目录名由 productName 决定，今后改名/换包名**不允许**改变 userData 路径（数据路径漂移 = 数据丢失）。
2. **迁移前备份**：仅当「db 文件存在 + 已有迁移已应用数 > 0 + 存在待应用迁移」时，复制 `yos.db` 到 `userData/backups/yos.db.bak-<ISO时间戳>.db`；滚动保留最近 **5** 份。
3. **备份失败 fail-open**：备份失败不阻止迁移（迁移本身单事务原子，是主保险；备份是第二道保险），仅结构化告警日志。
4. **迁移/初始化失败 fail-fast + 可见**：发布态 `createCatalogDatabase({ fallbackToMemory: false })` —— SQLite 打开失败直接抛错，**禁止静默降级内存库**；main 捕获后置 `initError`，经 `health:check` 暴露，渲染层显示明确错误横幅（不白屏、不假装正常）。
5. **不做**：完整 Backup/Restore UI、云备份、数据历史、自动修复、rollback 框架。迁移保持前向（forward-only）。
6. 当前「迁移前备份」≠「完整用户备份」；V1 再考虑完整备份、恢复、数据导出等能力。

## Alternatives

- 迁移前不备份。
- 完整备份/恢复系统（zip 导出导入、版本快照、回滚）。
- SQLite 打开失败静默降级内存库（现状）。

## Rejected

- **不备份**：升级事故无任何恢复点。
- **完整备份/恢复系统**：MVP 过度工程；备份触发点单一（迁移），先保住最高风险场景。
- **静默降级内存库**：数据看似消失、无任何提示，是最坏的用户体验与数据安全失败模式；保留内存降级仅用于「未指定 dbPath 的测试场景」。

## Why

以最小成本（一个 copy 操作 + 一个布尔参数）覆盖升级事故的恢复点；把「数据消失不可感知」变成「数据问题必可见」。复杂度与 MVP 边界匹配。

## Consequences

- `packages/database`：`createCatalogDatabase` 增加 `fallbackToMemory`（缺省 true 保测试兼容）；`SqliteCatalogAdapter` 暴露 `dbFilePath`；新增 `migration-backup.ts`；repository 构造时执行备份逻辑。
- desktop 组合根传 `fallbackToMemory: false`。
- 测试：数据库层新增备份/保留策略/损坏库 fail-fast 用例；1H-0 probe 验证真实升级场景与损坏场景 UI 反馈。
- 未来 V1 备份/恢复能力在本边界上扩展（备份目录约定已固定）。
- 「公开分发给陌生用户前，必须补充代码签名；自动更新依赖签名后再实施。」（发布决策备忘，与 ADR 009 一致）
