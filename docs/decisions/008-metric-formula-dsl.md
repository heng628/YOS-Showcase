# 008 — Metric Formula DSL（最小确定性子集）

- Date: 2026-08-24
- Status: **Accepted**
- 关联 Source of Truth: `docs/architecture/database-schema.md`（metric.definition）、`AGENTS.md` §12（确定性计算原则）、ADR 006（Metric 用户级）

## Context

Slice 1B 的 Metric 定义是 `formulaText` 字符串（AI 提案后经用户确认入库）。Slice 1C 需要把它变成「可确定性计算的公式」。若不限定语法边界，公式会演变为任意 SQL（注入/越权/不可验证）。

## Decision

定义**最小、明确、可验证的 Metric Formula DSL**（子集，不实现 SQL parser）：

```
expression := term (('+'|'-'|'*'|'/') term)*
term       := number | aggregate | '(' expression ')'
aggregate  := (SUM|AVG|MIN|MAX|COUNT) '(' field ')' | COUNT DISTINCT '(' field ')' | COUNT_DISTINCT '(' field ')'
field      := 与 DataAsset schema 真实字段名完全一致的标识符（支持中文）
```

- 执行链强制：`Formula → Parser → AST(typed) → Validator(字段/类型检查) → SQL Compiler → DuckDB`。
- 字段引用必须存在于 execution context 的 schema；不存在 → 拒绝。
- 类型规则：SUM/AVG 仅 Numeric（Number/Integer/Currency/Percentage）；MIN/MAX 允许 Numeric+Date/DateTime；COUNT/COUNT_DISTINCT 任意类型。
- 除法编译为 `x / NULLIF(y,0)`：除零 → null（确定性，不抛错）。
- SQL 只由 AST 编译生成（列名转义 + 必须命中 schema 映射）；**禁止字符串拼接任意 SQL**。
- 已知限制：字段名与函数关键字（SUM/AVG/MIN/MAX/COUNT/COUNT_DISTINCT）冲突时无法引用（tokenizer 大小写不敏感）；后续如需支持，引入引号字段名语法。

## Alternatives

- 完整 SQL parser / 允许任意 SQL。
- 直接把 formulaText 字符串拼进 SQL。
- 让 LLM 生成 SQL。

## Rejected

- **任意 SQL / 动态 SQL**：注入风险、越权、不可验证，违背「确定性程序计算」原则。
- **字符串拼接 SQL**：与上同理（即使字段名校验过也禁止，防止语法注入）。
- **LLM 生成 SQL**：违反 AGENTS.md §12 与 Agent 边界（AI 只负责理解与提案，不生成/执行 SQL）。
- 完整公式语言（JOIN/子查询/CTE/窗口函数）：超出 MVP，后续按需扩展 DSL 而非放开 SQL。

## Why

DSL 是「确定性计算」与「AI 提案」之间的安全契约：AI 只能按有限语法表达公式，程序解析并校验后才执行；类型检查保证 SUM(name) 之类的错误在 SQL 之前被拒绝。

## Consequences

- 实现位于 `packages/analytics`（parser/validator/compiler）；AST/Result 类型在 `packages/domain`。
- 1B 的 `formulaText` 格式继续作为 metric.definition 的存储格式（无需 schema 变更）。
- 指标定义不可执行时（语法/类型/字段缺失）返回结构化 Error，不落部分结果。
