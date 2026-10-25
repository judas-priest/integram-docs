# Module: agent-suggestions

**Path:** `src/api/v2/modules/agent-suggestions/`
**Files:** `router.js`, `service.js`, `clustering.js`, `sanitize.js`, `prompts.js`, `worker.js`
**Base URL:** `/api/v2/:db/agent-suggestions/...`
**Auth:** JWT required. GET endpoints require any authenticated role. POST (mutating) endpoints require `admin`.

## Purpose

HITL agent suggestions derived from behavioral patterns. The worker periodically scans swarm-memory behavioral logs, clusters similar patterns, drafts agent cards via LLM, and surfaces them as pending suggestions. Admins review, apply, or dismiss suggestions.

## Endpoints

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/agent-suggestions` | any | List suggestions (query: `status`, `limit`, `offset`) |
| GET | `/agent-suggestions/telemetry` | admin | Telemetry snapshot (counts by status) |
| GET | `/agent-suggestions/similar?q=...` | any | Search similar suggestions by text |
| GET | `/agent-suggestions/:id` | any | Get suggestion detail |
| POST | `/agent-suggestions/:id/apply` | admin | Start apply flow (preview draft) |
| POST | `/agent-suggestions/:id/apply/confirm` | admin | Confirm apply — updates status to 'applied' (atomic CAS) |
| POST | `/agent-suggestions/:id/dismiss` | admin | Dismiss suggestion |
| POST | `/agent-suggestions/:id/why-not` | admin | Reject with reason (body: `{ reason }`) |

All POST endpoints require CSRF token.

## Files

- **service.js** — business logic: CRUD, apply/dismiss lifecycle, telemetry, similarity search
- **router.js** — Express routes with Zod validation, role guards, CSRF
- **clustering.js** — greedy cosine clustering of behavioral patterns into candidate groups
- **sanitize.js** — PII scrubbing and ID anonymization before LLM processing
- **prompts.js** — LLM prompt templates and draft validator for agent card generation
- **worker.js** — setInterval periodic worker: `fetchCandidatePatterns` + `runOnce` pipeline

## Storage

- `_v2_agent_suggestions` (per-workspace table, auto-created via `ensureTables`)
  - Columns: `id`, `status` (pending/applied/dismissed/why_not/superseded), `cluster_id`, `draft` (JSONB — agent card draft), `patterns` (JSONB — source behavioral patterns), `rejection_reason`, `applied_agent_id`, `acted_by_user_id`, `acted_at`, `created_at`

## Worker

The worker runs on a configurable interval, scanning swarm-memory for new behavioral patterns. Environment variables:

| Env Var | Default | Description |
|---------|---------|-------------|
| `SUGGESTION_ENABLED` | `false` | Enable/disable the worker |
| `SUGGESTION_INTERVAL_MS` | `86400000` | Scan interval (24 hours default) |
| `SUGGESTION_MIN_CLUSTER_SIZE` | `3` | Minimum patterns to form a cluster |
| `SUGGESTION_PER_RUN_LIMIT` | — | Max suggestions per run |
| `SUGGESTION_BATCH_SIZE` | — | Batch size for processing |
| `SUGGESTION_MIN_OBSERVATIONS` | — | Min observations to form a suggestion |
| `SUGGESTION_MIN_CONFIDENCE` | — | Min confidence threshold |
| `SUGGESTION_SIMILARITY_THRESHOLD` | — | Cosine similarity threshold for dedup |
| `SUGGESTION_COOLDOWN_DAYS` | — | Days before re-suggesting similar |
| `SUGGESTION_RETIREMENT_THRESHOLD` | — | Threshold for retiring old suggestions |

## Dependencies

- `modules/swarm-memory/` — behavioral pattern source
- `modules/agent-registry/` — target for applied suggestions
- LLM service — drafts agent cards from clustered patterns
- Event bus: none
