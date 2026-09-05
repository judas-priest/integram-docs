# Module: swarm-memory

**Path:** `src/api/v2/modules/swarm-memory/`
**Files:** Many — see below.
**Base URL:** `/api/v2/:db/swarm-memory/...`
**Auth:** JWT required for all endpoints.

## Purpose

Persistent AI agent long-term memory. Stores facts, procedures, anti-patterns, and behavioral observations as vector-indexed entries. Hybrid search: semantic (pgvector HNSW) + full-text (tsvector) + recency + importance scoring. Also includes Knowledge Augmented Graph (KAG) for entity/relationship knowledge.

For detailed documentation see [`docs/archive/swarm-memory.md`](../../../docs/archive/swarm-memory.md).

## Key Submodules

| File | Description |
|------|-------------|
| `service.js` | Core memory CRUD: write, recall, forget, upvote/downvote, compact, decay, contradictions, extract-insights, reflect-on-failure |
| `kag.js` | Knowledge Augmented Graph: import ontology/entities/relations, search |
| `procedures.js` | Procedure storage and downvote support |
| ~~`mcp-adapter.js`~~ | Removed — memory tools are exposed directly via TOOL_DEFS in `agent/index.js` |
| `behavioral-extractor.js` | Extracts behavioral patterns and preferences from interaction history |
| `behavioral-collector.js` | Listens to agent events and collects behavioral signals for analysis |
| `context-compression.js` | Compresses long conversation context for memory extraction while preserving key facts |
| `router.js` | HTTP endpoints (mounted at `/:db/swarm-memory/`) |

## Endpoints

### Core memory

| Method | Path | Role | Description |
|--------|------|------|-------------|
| POST | `/swarm-memory/write` | editor | Store/update a memory entry (`key`, `value`, `tags`, `scope`) |
| GET | `/swarm-memory/list` | any | List memory entries (`?tag=`, `?includeShared=`, `?limit=`) |
| GET | `/swarm-memory/recall` | any | Semantic + FTS recall (`?question=`) |
| DELETE | `/swarm-memory/forget` | editor | Delete a memory entry by key (body: `{ key }`) |
| GET | `/swarm-memory/shared` | any | List shared-scope memories |
| POST | `/swarm-memory/upvote` | editor | Mark memory as helpful (raises relevance_score) |
| POST | `/swarm-memory/downvote` | editor | Mark memory as irrelevant (lowers relevance_score) |
| GET | `/swarm-memory/guardrails` | any | Negative anti-pattern rules for prompt injection |
| GET | `/swarm-memory/search` | any | Hybrid search GET (`?q=`, `?limit=`, `?lambda=`) |
| POST | `/swarm-memory/search` | any | Hybrid search POST (body: `{ q, limit, lambda, filters }`) |
| GET | `/swarm-memory/history` | any | Bi-temporal change history for a key (`?key=`) |
| GET | `/swarm-memory/stats` | any | Memory stats (admin can override agentId) |
| GET | `/swarm-memory/contradictions` | any | List contradictory memories (`?status=unresolved\|all`) |
| POST | `/swarm-memory/contradictions/resolve` | editor | Resolve a contradiction |
| POST | `/swarm-memory/contradictions/auto-resolve` | admin | Trigger deterministic auto-resolution (newer `updated_at` wins, `relevance_score` tiebreaker — no LLM involved) |

### Admin / maintenance

| Method | Path | Role | Description |
|--------|------|------|-------------|
| POST | `/swarm-memory/compact` | admin | LSM compaction (merge similar memories) |
| POST | `/swarm-memory/decay` | admin | Apply time-based relevance_score decay |
| GET | `/swarm-memory/audit` | admin | Audit log of all memory operations |
| POST | `/swarm-memory/expire-negatives` | admin | Expire stale anti-pattern rules (>30 days) |
| POST | `/swarm-memory/probe-negatives` | admin | Counterfactual probe for negative rules |

### Agent utilities

| Method | Path | Role | Description |
|--------|------|------|-------------|
| POST | `/swarm-memory/extract-insights` | editor | Send session history for LTM extraction (fire-and-forget) |
| POST | `/swarm-memory/reflect-on-failure` | editor | Self-correction hint for a failed tool call |
| POST | `/swarm-memory/recall-utility` | editor | Log post-session recall utility signal |
| GET | `/swarm-memory/ping` | any | Healthcheck + auth validation |

### Shared State (event-sourced KV)

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/swarm-memory/state` | any | List all shared-state keys |
| GET | `/swarm-memory/state/:key` | any | Get single key value |
| PUT | `/swarm-memory/state/:key` | editor | Set key value |
| DELETE | `/swarm-memory/state/:key` | editor | Delete key |
| GET | `/swarm-memory/state/:key/log` | admin | Event log for key |

### KAG (Knowledge Augmented Graph)

| Method | Path | Role | Description |
|--------|------|------|-------------|
| POST | `/swarm-memory/kag/import-ontology` | admin | Import class hierarchy |
| POST | `/swarm-memory/kag/import-entities` | admin | Bulk import entities |
| POST | `/swarm-memory/kag/import-relations` | admin | Bulk import relations |
| DELETE | `/swarm-memory/kag/data` | admin | Delete KAG data (`?source=`) |
| GET | `/swarm-memory/kag/stats` | admin | KAG statistics |
| POST | `/swarm-memory/kag/sync-memory-files` | admin | Sync KAG from memory files on disk |
| GET | `/swarm-memory/kag/search` | any | Search entities by text (`?q=`, `?limit=`, `?lambda=`) |
| POST | `/swarm-memory/kag/edges` | any | Bulk fetch KAG relations and missing neighbor entities (body: `{ ids: number[] }`) |
| GET | `/swarm-memory/kag/traverse` | any | Graph traversal from a starting entity (`?id=`, `?depth=`) |
| GET | `/swarm-memory/kag/clusters` | any | Detect entity clusters in the knowledge graph |
| GET | `/swarm-memory/kag/anomalies` | any | Detect anomalies in the knowledge graph |

## Hybrid Search Algorithm

Combines two signals with MMR diversity reranking:

1. **Vector similarity** — pgvector cosine distance (weight: 0.35)
2. **Full-text search** — tsvector GIN index BM25 (weight: 0.20)
3. **Recency** — exponential decay `exp(-age_days / 30)` (weight: 0.15)
4. **Relevance score** — upvote/downvote-adjusted score (weight: 0.15)
5. **Entity boost** — KAG entity match bonus (weight: 0.15)
5. **MMR reranking** — `lambda` param (default 0.7): 1.0 = pure relevance, 0.0 = pure diversity

Configurable via `?lambda=` query param or body field.

## Contradiction Detection

Contradictions are automatically detected via embedding similarity during `write`. Auto-resolution is purely deterministic: newer `updated_at` wins, with `relevance_score` as tiebreaker — no LLM involved. Manual resolve via `POST /swarm-memory/contradictions/resolve`.

## DB Tables (global public schema)

- `agent_memory` — `key`, `agent_id`, `workspace_db`, `value` (JSONB), `tags`, `embedding` (vector), `tsvector`, `relevance_score`, `scope`, `created_at`, `accessed_at`
- `agent_procedures` — step-by-step recipes extracted from interactions
- `shared_state` — event-sourced KV store for cross-agent shared knowledge
- `shared_state_log` — audit of shared_state changes
- `memory_edges` — similarity relationships between memory entries (A-MEM Zettelkasten)
- `memory_contradictions` — pairs of contradicting memories with resolution status
- `memory_audit` — change log (write/forget/compact operations)
- `kag_entities` — knowledge graph entities with embeddings and class assignments
- `kag_classes` — entity type hierarchy (ontology)
- `kag_relations` — relationships between entities with type and properties
