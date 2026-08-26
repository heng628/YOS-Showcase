# 009 — LLM 凭据本地安全存储（safeStorage）

- Date: 2026-08-24
- Status: **Accepted**
- 关联 Source of Truth: `technical-stack-adr.md`（AI Client 边界）、`context-manifest.md`（云端能力面）、AGENTS.md §26/§31

## Context

MVP 的 LLM 凭据最初仅经环境变量（`YOS_LLM_BASE_URL/MODEL/API_KEY`）注入。该方案对开发者成立，对普通用户不成立（不会设置环境变量），且 `.env` 文件易被误复制/误传。发布前必须提供面向用户的凭据配置，同时保证凭据机密性：key 永不进入 renderer、preload 返回值、SQLite、Activity Log、普通日志。

## Decision

1. **存储**：用户在设置界面输入 endpoint/model/key → 仅 Electron main 经 `safeStorage.encryptString` 加密 → 密文（base64）写入 `userData/llm-config.json`（`{baseUrl, model, keyEncrypted}`）。**明文永不落盘**；解密仅在 main 构建 LLMClient 时进行（内存中）。
2. **边界**：key 永不进入 renderer/preload/SQLite/Activity Log/日志；`llm:getConfig` 只返回 `{hasKey, baseUrl, model, source, encryptionAvailable}`。
3. **优先级**：`YOS_LLM_MOCK=1` > 存储配置 > 环境变量（env 保留为开发/高级选项，二者并存时存储优先）。
4. **降级**：`safeStorage.isEncryptionAvailable() === false` 时**拒绝持久化 key**（不落明文文件）；允许本次会话内存使用，UI 明示「仅本次会话有效」。
5. renderer 永不直连 LLM provider；调用链不变（renderer → IPC → main → LLMClient）。

## Alternatives

- key 明文写入配置文件。
- key 存入 SQLite catalog。
- key 由 renderer 持有并直连 provider。
- 引入 OS keychain 第三方库（keytar）。

## Rejected

- **明文配置/存 SQLite**：泄露面大（备份/日志/同步都会复制）；SQLite 是业务数据权威，凭据不属于业务数据。
- **renderer 持有/直连**：违反最小暴露原则；攻击面扩大（CSP 不能保护内存中的 key）。
- **keytar**：引入原生依赖 + 跨平台问题；Electron 自带 `safeStorage`（Windows=DPAPI）足够 MVP。

## Why

safeStorage 是 Electron 官方 OS 级加密（Windows DPAPI），零新增依赖；凭据与业务数据（本地权威）隔离，为未来云同步/云端能力面留出干净边界。

## Consequences

- 新增 `apps/desktop/src/main/llm-config.ts`（唯一凭据读写模块）；renderer 只有 `hasKey` 认知。
- 配置优先级固化：mock > stored > env。
- 未来 V1 接账号/订阅时，凭据迁移到云端能力面授权 scope；本地缓存策略沿用本 ADR 边界。
- 「公开分发给陌生用户前，必须补充代码签名；自动更新依赖签名后再实施。」（发布决策备忘）
