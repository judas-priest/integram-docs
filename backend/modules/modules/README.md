# Backend Modules — Documentation Index

All modules live in `src/api/v2/modules/`. Each module has a `router.js` (Express routes) and usually a `service.js` (business logic). Some modules have additional subfiles for workers, listeners, or sub-features.

## Module List

### Auth and Users

| Module | File | Description |
|--------|------|-------------|
| [iam](iam.md) | `modules/iam/` | Global auth: register, login, JWT refresh, password reset, magic links |
| [workspaces](workspaces.md) | `modules/workspaces/` | Workspace CRUD, members, invitations, bots, service keys, templates, clone, remote database (separate PG server per workspace) |
| [orgs](orgs.md) | `modules/orgs/` | Organizations — group workspaces under a company/team |
| [admin](admin.md) | `modules/admin/` | Workspace admin: backup/restore, grants, roles, row rules, DLQ, BKI |
| [desktop](desktop.md) | `modules/desktop/` | Persistent user browser tabs (global, no `:db`) |

### Data Core

| Module | File | Description |
|--------|------|-------------|
| [objects](objects.md) | `modules/objects/` | EAV object CRUD, bulk ops, history, CSV import, trash, record sharing |
| [schema](schema.md) | `modules/schema/` | DDL: types (tables) and requisites (columns), computed columns, audit, snapshots |
| [lookups](lookups.md) | `modules/lookups/` | Dropdown option lists for reference columns |
| [views](views.md) | `modules/views/` | Saved view configs (filters, sort, columns, view type), public sharing |
| [computed-reqs](computed-reqs.md) | `modules/computed-reqs/` | LOOKUP, ROLLUP, FORMULA virtual column definitions |
| [templates](templates.md) | `modules/templates/` | Record templates — pre-filled field values |

### AI

| Module | File | Description |
|--------|------|-------------|
| [ai](ai.md) | `modules/ai/` | Multi-agent chat (SSE), 24 agents, 530+ tools (platform + per-workspace custom), autofill, AI columns, MCP, HITL |
| [swarm-memory](swarm-memory.md) | `modules/swarm-memory/` | Persistent AI memory: hybrid search (vector+FTS), behavioral collector, KAG |
| [normalizer](normalizer.md) | `modules/normalizer/` | AI document normalization pipeline: classify → extract → HITL → populate |
| [agent-registry](agent-registry.md) | `modules/agent-registry/` | Registry of external AI agents for delegation |
| [agent-suggestions](agent-suggestions.md) | `modules/agent-suggestions/` | HITL agent suggestions from behavioral patterns |
| [workspace-tools](workspace-tools.md) | `modules/workspace-tools/` | Per-workspace custom tool registry: JS functions in isolated-vm sandbox, capability-based bridges (query/fetch/ai/write/delete), MCP + debate engine integration |

### Documents and Files

| Module | File | Description |
|--------|------|-------------|
| [documents](documents.md) | `modules/documents/` | Block-based document editor, real-time collab, versioning, folders, tags, export |
| [files](files.md) | `modules/files/` | File upload/download (50 MB), PDF/DOCX parsing, OCR, embedding |

### Automations and Integrations

| Module | File | Description |
|--------|------|-------------|
| [automations](automations.md) | `modules/automations/` | Event-driven rules: 12 trigger types, BullMQ, cron, AI insights |
| [webhooks](webhooks.md) | `modules/webhooks/` | Outgoing HTTP notifications, HMAC signatures, retry, delivery log |
| [connectors](connectors.md) | `modules/connectors/` | External API integrations, field mapping, incremental sync, CDEK |
| [http-button](http-button.md) | `modules/http-button/` | HTTP Button column (type 1016): direct HTTP fetch from cell, no LLM, placeholder interpolation |
| [script-button](script-button.md) | `modules/script-button/` | Script Button column (type 1020): user-written JS runs in Node.js vm sandbox on click, row data + fetch available |
| [browser](browser.md) | `browser/` (отдельный процесс) | Headless Playwright сервис (порт 3099): `browse()` в run_script, `search_prices` AI-тул |
| [notifications](notifications.md) | `modules/notifications/` | In-app notification feed, unread count, approval actions |
| [comments](comments.md) | `modules/comments/` | Object comment threads with emoji reactions |

### Analytics and Reports

| Module | File | Description |
|--------|------|-------------|
| [reports](reports.md) | `modules/reports/` | SQL report builder: aggregation, joins, filters, CSV export, SSE stream (formula validation at write-time) |
| [dashboards](dashboards.md) | `modules/dashboards/` | Widget dashboard configs (JSONB) |
| [timeseries](timeseries.md) | `modules/timeseries/` | Time-series data ingestion and aggregated queries |
| [audit-export](audit-export.md) | `modules/audit-export/` | Unified audit log export (objects + schema + reports) |

### Portal and Telegram

| Module | File | Description |
|--------|------|-------------|
| [portal](portal.md) | `modules/portal/` | Customer portal: OTP/Telegram auth, catalog, cart, orders, AI chat, Nuxt SSR proxy |
| [telegram-bots](telegram-bots.md) | `modules/portal/` (bot mgmt) | Multi-bot Telegram constructor: config, commands, reactions via automations, webhook |

### Realtime

| Module | File | Description |
|--------|------|-------------|
| [ws](ws.md) | `api/v2/ws.js` | WebSocket server: 12 frame types, 4 subscription channels (`objects`, `documents`, `sandbox-collab`, `point`), subscribe-time authorization, document delta persistence |
| [calls](calls.md) | `modules/calls/` | Pure P2P 1-1 voice/video + workspace voice room (≤8), TURN HMAC creds, WebRTC signalling |
| [presence](presence.md) | `modules/presence/` | Durable log of "who was on this work point and what they did" (`_v2_presence_log`, 90-day retention), fed by the WS `point` channel. **Not** the member's "last visit" — that is `_v2_memberships.last_seen_at`, see [ADR-026](../../../docs/adr/026-presence-and-last-seen.md) |
| [tts](tts.md) | `modules/tts/` | Text-to-speech synthesis via Piper TTS |

### Knowledge Graph

| Module | File | Description |
|--------|------|-------------|
| [graph](graph.md) | `modules/graph/` | PostgreSQL-backed graph (nodes+edges), vector search, agent memory REST API |

### Communication

| Module | File | Description |
|--------|------|-------------|
| [teamchat](teamchat.md) | `modules/teamchat/` | Internal workspace chat: rooms, topics, messages, members |
| [decisions](decisions.md) | `modules/decisions/` | Architectural decisions with typed links, discussions, vector search |
| [meta-kb](meta-kb.md) | `modules/meta-kb/` | Debate-based knowledge curation: parallel expert opinions + LLM synthesis before decision fixation |

### Project Management

| Module | File | Description |
|--------|------|-------------|
| [pm](pm.md) | `modules/pm/` | Issues, sprints, board (Kanban), backlog, metrics (velocity, burndown, cycle time), org-level aggregation |
| [nightcall](nightcall.md) | `modules/nightcall/` | Specification-driven execution: requirements, intents, evidence, compliance/release decisions, DAG generation |

### Security

| Module | File | Description |
|--------|------|-------------|
| [dlp](dlp.md) | `modules/dlp/` | Data Loss Prevention — policies for data protection |

### Developer Tools

| Module | File | Description |
|--------|------|-------------|
| [codespace](codespace.md) | `modules/codespace/` | Git hosting: Smart HTTP, repos, branches, pull requests (500 MB limit, Docker sandbox for CI gate) |
| [forms](forms.md) | `modules/forms/` | Public data collection forms with unique tokens |
| [testing](testing.md) | `modules/testing/` | Internal QA test session tracking |
| [specs](specs.md) | `modules/specs/` | Declarative executable specs (invariants) |
| error-collector | `modules/error-collector/` | Frontend/backend error collection with fingerprint dedup. **No module doc yet** — the link here pointed at a file that has never existed |
| [resolution](resolution.md) | `modules/resolution/` | Golden record resolution (MDM) |

## Deprecated / Stub

| Module | Status | Notes |
|--------|--------|-------|
| `auth` | Deprecated | Replaced by `iam` |
| `entity-meta.js` | Stub/redirect | Entity metadata now in `schema/type-meta.js` |
| `computed-reqs` | Thin wrapper | Logic lives in `utils/computed-reqs.js` |
