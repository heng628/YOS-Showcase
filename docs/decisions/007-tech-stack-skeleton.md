# 007 — 技术栈骨架：Electron 42 + SQLite/DuckDB 双存储 + pnpm/Turbo

- Date: 2026-08-23
- Status: **Accepted**
- 关联 Source of Truth: `docs/architecture/technical-stack-adr.md`、`README.md`

## Context

0 代码状态下为「本地权威 + 确定性分析 + AI 客户端 + 后台能力」选型；环境无 MSVC（不能源码编译 native 模块），必须走预编译二进制。

## Decision

- Desktop：**Electron 42**（contextIsolation/sandbox；DB 只在 main）
- 存储：**SQLite（better-sqlite3@12.11.1）+ DuckDB（@duckdb/node-api，N-API）** 分工（catalog vs 记录/分析）
- better-sqlite3 **双 ABI**：Node ABI → `build/Release`；Electron ABI → `build/electron`（main 经 `nativeBinding` 显式加载）
- Monorepo：**pnpm 10 + Turborepo**；ARCHITECTURE GUARD（本地 ESLint 规则按说明符拦截 + no-restricted-paths + pnpm 隔离）
- 前端 React 19 + Vite；AI 薄 LLMClient 接口；Vitest + Playwright(Electron) + CI

## Alternatives

- Tauri（Rust + WebView）；Electron 43 + better-sqlite3@12.12.0；IndexedDB；单库（SQLite-only / DuckDB-only）；Nx；npm/yarn workspaces。

## Rejected

- **Tauri**：MVP 需 Node 生态（文件/DB/AI/调度），全栈 TS 单一语言迭代最快；体积不是 MVP 瓶颈（V1+ 可再评估，领域层 TS 可复用）。
- **Electron 43**：better-sqlite3 无 ABI 148 的 npm 预编译（12.12.0 未发布 npm；12.11.1 最高含 ABI 146=Electron 42）。
- **better-sqlite3@12.12.0**：GitHub 有 release 但 npm 未发布该版本。
- **IndexedDB**：不是权威存储载体，无 SQL/版本/同步能力。
- **单库**：SQLite-only 拖累 Analytics；DuckDB-only 做不了系统目录 CRUD。
- **Nx / npm-workspaces**：Nx 过重；npm 依赖边界松散（pnpm 严格隔离天然强制边界）。

## Why

单一 TS 语言 + 本地权威合理载体 + 边界可强制（pnpm 隔离 + GUARD + Electron 安全基线）+ AI 不入侵 Domain + 不引入 MSVC 编译依赖。

## Consequences

- `pnpm.onlyBuiltDependencies` 白名单必须配置（electron/esbuild/better-sqlite3/@duckdb/node-api）。
- 双 ABI 依赖 `rebuild:sqlite:electron` 脚本；重装依赖后需重跑（README 已注明）。
- **Drizzle ORM 已选型但尚未引入**：骨架阶段用 better-sqlite3 直连接口；业务 schema 阶段再引入 Drizzle（非冲突，属阶段性落地）。
- Electron 二进制下载需网络（可用 `ELECTRON_MIRROR` 镜像）。
