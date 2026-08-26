# YOS · AI Data-Asset Workspace

> *[中文版](README.md)*

> **Turn your own data into an asset you own — the program computes it precisely, AI understands and explains it, anomalies get watched, reports come out in one tap, and the system knows you better over time.**

YOS is a **local-first** personal AI data workspace (Electron desktop app). It is neither another dashboard tool nor just a chatbot — its core is treating **data assets as the root**, paired with AI understanding, analysis, monitoring, and reporting, forming a complete loop:

```
Data Asset → AI Understanding → Analysis → Monitoring → Suggestions → User Decision → More Data
```

> This repo showcases product & engineering quality: the core source & data pipeline are private; product definition and architecture designs are open (see `docs/`). Contact the author for a code review during recruitment.

---

## Why YOS

E-commerce operators, content creators, freelancers, learners... everyone has their own data every day, scattered across Excel files, platform dashboards, and accounting apps. When they want to "understand, watch, and summarize," they usually face three options:

1. **Learn BI (Superset/Metabase and friends)** — requires deploying servers, connecting databases, and learning chart configuration; the bar is far above personal users;
2. **Ask an AI once (ChatGPT/Claude uploads)** — the answer is forgotten the moment the session ends; nothing becomes an asset;
3. **Keep using Excel** — numbers are computable, but nothing is understood, watched, or accumulated.

**YOS takes a fourth path**: turn data into **reusable assets**, let AI understand and monitor within **your authorized scope**, and keep your data on your own machine, always.

---

## Core Features

| Feature | Description |
|---|---|
| 📦 **Data Assets** | CSV/Excel import or manual entry; field type/semantic inference + quality check; assets are **user-level shared** — one original, referenced (mounted) by any workspace, never duplicated |
| 🧠 **AI Understanding (human-confirmed)** | AI proposes field semantics and metrics (Proposals) → **activated only after your confirmation**; AI never modifies data directly |
| 📐 **Deterministic Metrics** | Formula DSL (SUM/AVG/MIN/MAX/COUNT + arithmetic) computed by a **program engine** (DuckDB); LLM never computes or generates SQL — accurate and reproducible |
| 🗂️ **Spaces / Workspaces / Modules** | Work/Study/Personal scene groups; modules (KPI / Chart / Table) show real values on cards |
| 🤖 **AI Analysis / Command Center** | Ctrl+K opens the command center; natural-language intent routing to read-only analysis (evidence-based, never fabricated) |
| 🔔 **Monitoring** | Rule (metric + scope + condition + sensitivity) → proposal confirm → deterministic evaluation → event trigger → user resolve; Mute/Pause/Delete semantics strictly separated |
| 📄 **Reports** | One-tap temporary report (Markdown preview + export) as the output layer |
| 🛡️ **Trust by Design** | All writes go through `Proposal → Confirmation → Execute → Result → Activity Log`; **deleting raw data is never automated**; data stays local and migratable |

---

## Screenshots

<table>
  <tr>
    <td align="center"><img src="assets/screenshots/01-sidebar-collapsed.png" width="100%"/><br/><sub>Dark icon rail (76px, collapsed)</sub></td>
    <td align="center"><img src="assets/screenshots/02-assets-list.png" width="100%"/><br/><sub>Data Assets page (real data: 12 rows)</sub></td>
  </tr>
  <tr>
    <td align="center"><img src="assets/screenshots/03-ai-understanding.png" width="100%"/><br/><sub>AI understanding → field semantics + metric suggestions (7 proposals)</sub></td>
    <td align="center"><img src="assets/screenshots/04-activated-metric.png" width="100%"/><br/><sub>Confirmed activation → metric enters the library</sub></td>
  </tr>
  <tr>
    <td align="center"><img src="assets/screenshots/05-sidebar-hover-wave.png" width="100%"/><br/><sub>On hover: active wave follows + icon scales up</sub></td>
    <td align="center"><img src="assets/screenshots/06-sidebar-expanded.png" width="100%"/><br/><sub>300ms dwell → expands to 200px (overlay, no layout shift)</sub></td>
  </tr>
  <tr>
    <td align="center"><img src="assets/screenshots/07-module-cards.png" width="100%"/><br/><sub>Space overview: module cards show real KPI (44)</sub></td>
    <td align="center"><img src="assets/screenshots/08-module-peek.png" width="100%"/><br/><sub>Quick preview: KPI card (Total Qty = 44)</sub></td>
  </tr>
  <tr>
    <td align="center"><img src="assets/screenshots/09-dashboard-kpi.png" width="100%"/><br/><sub>Dashboard (metric card: deterministic result)</sub></td>
    <td align="center"><img src="assets/screenshots/10-report-preview.png" width="100%"/><br/><sub>Temporary report (Markdown preview)</sub></td>
  </tr>
  <tr>
    <td align="center"><img src="assets/screenshots/11-monitoring-proposal.png" width="100%"/><br/><sub>Monitoring rule Proposal → user confirmation</sub></td>
    <td align="center"><img src="assets/screenshots/12-monitoring-triggered.png" width="100%"/><br/><sub>Anomaly triggered (44 < 4000) → [Triggered]</sub></td>
  </tr>
  <tr>
    <td align="center"><img src="assets/screenshots/13-monitoring-resolved.png" width="100%"/><br/><sub>User resolve → [Resolved] (fully traceable)</sub></td>
    <td></td>
  </tr>
</table>

> Screenshots are from real runs: 12-row retail-sales CSV → AI understanding (7 proposals) → confirmed activation → metric `Total Qty = SUM(Qty) = 44` → module/Dashboard → monitoring trigger & resolve. UI motion (wave/expand) captured with measured timing.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Renderer (React 19 + Vite)                              │
│  Sidebar(76px↔200px overlay) · SpaceView · Dashboard ·   │
│  Monitoring · Report                                     │
└──────────────────────┬──────────────────────────────────┘
                       │ preload IPC (contextIsolation)
┌──────────────────────▼──────────────────────────────────┐
│  Main (Electron 42)   ── composition root / IPC validation│
│  UseCases: import·understand·confirm·metric·analyze·       │
│            monitoring·report                              │
│  Agent: intent routing (read-only whitelist, zero writes) │
├─────────────────────────────────────────────────────────┤
│  Domain / Application  (rules · state machines ·          │
│                         deterministic formulas · Proposal)│
├─────────────────────────────────────────────────────────┤
│  SQLite (catalog/entities)  +  DuckDB (records/analytics) │
│  Local authority · no cloud · migratable data             │
└─────────────────────────────────────────────────────────┘
              ▲ OpenAI-compatible LLM (understanding only)
```

**Stack**

| Layer | Choice |
|---|---|
| Desktop runtime | Electron 42 · contextIsolation · DB only in main |
| Frontend | TypeScript (strict) · React 19 · Vite |
| Storage | better-sqlite3 + Drizzle (catalog) · DuckDB (records + deterministic analytics) |
| Tooling | pnpm workspaces + Turborepo · Vitest (201 tests) · Playwright E2E |
| Quality | ESLint architecture guard (blocks agent→database cross-layer imports) + isolation |
| AI | Own thin LLMClient abstraction (OpenAI-compatible) + usage metering; mock injection for full-chain verification |

**Key engineering designs (see `docs/`)**

- `docs/architecture/domain-model.md` — data asset = user-level shared asset (one original, Mount reuse, delete-proof)
- `docs/architecture/agent-architecture.md` — call chain `Agent → Tool → Application → Domain → Repo → Data`; agent never touches DB directly
- `docs/architecture/permission-model.md` — permissions L0–L3 × confirmation policy A/B/C/D (raw-data deletion = never automated)
- `docs/architecture/state-model.md` — orthogonal three-axis state machine (Lifecycle × Execution × Health)
- `docs/memory/decisions/` — 11 ADRs (each documents why alternatives were rejected)
- `docs/product/PRD.md` — product requirements (core loop / MVP scope / V1-V2 roadmap)
- `docs/design/design-system.md` — design tokens & visual system

---

## Engineering Highlights

- **Architecture guard**: pnpm dependency isolation + custom ESLint rules enforcing layer boundaries (an agent→database import fails the build)
- **Verification baseline**: `pnpm verify` (typecheck + lint + **201 tests** + build) all green; E2E smoke passing
- **Full-chain verification with real data**: import → understand (7 proposals) → confirm → metric = 44 → dashboard → monitoring → report, **14/14 assertions pass** (automated probe, re-runnable)
- **Security hardening** (ADR 009–011): CSP, IPC sender + runtime payload validation, safeStorage secret storage, DuckDB fail-fast (no silent memory fallback), pre-migration auto backup
- **Process & versioning**: product-discussion / engineering-execution role split; safe versions `safe-v1/v2/v3` (commit + tag + rollback guide)

---

## Run (dev preview)

```bash
pnpm install
pnpm dev            # Electron dev mode
# First run: configure an OpenAI-compatible LLM, or demo mode:
YOS_LLM_MOCK=1 pnpm dev   # deterministic Mock LLM (full chain, no API key needed)
```

> All data stays local (`yos.db + records/*.duckdb`); import/analysis/monitoring/report work offline (only LLM capability calls need network).

---

## Roadmap

1. **S1 Validate** (current): single-machine MVP with seed users — real-data closed loop
2. **S2 Capability surface**: cloud sync/backup/metering, more data sources (API/OAuth), formal report center, **Agent memory & learning** (deterministic, evidence-driven personal profile — user-visible and erasable), agent write capability (propose → confirm → execute)
3. **S3 Expand**: small-team collaboration; establish the "personal data asset" mindset

---

> Full source (monorepo: 9 packages, 201 tests, E2E & probe tooling) lives in the author's private repository; contact GitHub [@heng628](https://github.com/heng628) for a code review.
