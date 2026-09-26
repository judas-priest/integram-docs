# Decisions Module

Module for managing architectural decisions, their relationships, and discussions.

**Path:** `src/api/v2/modules/decisions/`
**Files:** `router.js`, `service.js`, `listeners.js`, `kag-index.js`

## Tables

| Table | Description |
|-------|-------------|
| `_v2_dc_decisions` | Decision records with title, domain, verdict, embedding, metadata |
| `_v2_dc_links` | Directed typed links between decisions (compatible, conflicts, parent) |
| `_v2_dc_decision_history` | Change history: `id`, `decision_id`, `field`, `old_value`, `new_value`, `changed_by`, `changed_at` |

## API Endpoints

All endpoints require JWT auth and are scoped to `/:db`.

### Decisions

| Method | Path | Description |
|--------|------|-------------|
| GET | `/decisions` | List with filters (domain, verdict, q, team, sort, cursor) |
| POST | `/decisions` | Create decision (optionally with linked chat room) |
| GET | `/decisions/:id` | Get decision detail + links + discussions |
| PATCH | `/decisions/:id` | Update decision (verdict only by humans) |
| DELETE | `/decisions/:id` | Delete decision |

### Links

| Method | Path | Description |
|--------|------|-------------|
| GET | `/decisions/:id/links` | List links grouped by rel_type |
| POST | `/decisions/:id/links` | Create link (UNIQUE on from_id, to_id, rel_type) |
| DELETE | `/decisions/:id/links/:linkId` | Remove link |

### Iterations

| Method | Path | Description |
|--------|------|-------------|
| GET | `/decisions/:id/iterations` | List Meta KB iterations linked to this decision |

### Graph

| Method | Path | Description |
|--------|------|-------------|
| GET | `/decisions/graph` | Get decision graph (nodes + edges for all decisions) |

### History

| Method | Path | Description |
|--------|------|-------------|
| GET | `/decisions/:id/history` | Get change history for a decision (any role) |

### Conflicts

| Method | Path | Description |
|--------|------|-------------|
| GET | `/decisions/:id/conflicts` | Analyze conflicts: direct, transitive, KAG, memory. Returns `{ directConflicts, transitiveConflicts, kagConflicts, memoryConflicts, resourceConflicts, recommendation }` |
| GET | `/decisions/:id/kag-stats` | Return KAG entity/relation counts for a decision |

### Discussions

| Method | Path | Description |
|--------|------|-------------|
| GET | `/decisions/:id/discussions` | Topics from linked chat room |

## Verdict values

`proposed`, `accepted`, `rejected`, `superseded`, `draft`

## Service exports

- `ensureTables(pool, db)` — schema + indexes
- `listDecisions(pool, db, opts, user)` — cursor pagination, hybrid vector+FTS search. FTS uses OR semantics (`plainto_tsquery` with `& → |`) so multi-word queries match decisions containing ANY term, not all
- `createDecision(pool, db, body, user)` — INSERT + optional chat room
- `getDecision(pool, db, id, user)` — decision + links + discussions
- `updateDecision(pool, db, id, body, user)` — agent verdict lock
- `deleteDecision(pool, db, id, user)`
- `listLinks(pool, db, decisionId, user)`
- `createLink(pool, db, decisionId, body, user)`
- `deleteLink(pool, db, decisionId, linkId, user)`
- `getDiscussions(pool, db, decisionId, user)`
- `getHistory(pool, db, decisionId)` — change history for a decision
- `analyzeDecisionConflicts(pool, db, decisionId)` — conflict analysis across decisions
- `listIterationsByDecision(pool, db, decisionId)` — Meta KB iterations linked to a decision
- `getDecisionGraph(pool, db)` — nodes + edges for all decisions

## Cross-module dependencies

- Imports `createRoom` and `requestApproval` from `../teamchat/service.js`

### Agent Verdict Approval (`requestApproval`)

When `updateDecision()` receives a verdict change from a user whose `username` starts with `agent:`, the change is **not applied directly**. Instead:

1. `requestApproval(pool, db, topicId, { type: 'change_verdict', params: { decisionId, verdict, roomId } })` is called
2. A pending approval record is created in `_v2_automation_suspended`
3. All admin members of rooms containing the topic (or the decision's chat room) are notified
4. The function returns `{ data: { id, status: 'approval_requested', suspendedId } }`
5. A human admin must approve/reject via `POST /teamchat/topics/:topicId/approvals/:actionId`

This ensures agents cannot unilaterally change decision verdicts — human oversight is always required.

- Reads teamchat tables (`_v2_tc_rooms`, `_v2_tc_topics`, `_v2_tc_room_topics`, `_v2_tc_messages`) for discussions
- Uses `embedText` from `../../utils/embedding-sync.js` for vector search

## KAG / Memory Conflict Analysis

The decisions module integrates with the Knowledge-Augmented Generation (KAG) and agent memory system:

1. **On decision creation** — the anamnesis listener queries the 5 most similar decisions by vector cosine distance. Found conflicts are posted as a summary message to the linked teamchat room.
2. **Conflict analysis** — `analyzeDecisionConflicts()` performs direct, transitive, KAG and memory conflict detection and **returns the conflict data**. The `decision.created` listener in `listeners.js` is responsible for posting a summary to the linked teamchat room. **No auto-links** are created — conflict links (`rel_type='conflicts'`) must be created manually via `POST /decisions/:id/links`.
3. **Search** — `GET /decisions?q=...` performs hybrid vector+FTS search (cursor-based pagination, OR semantics for FTS).

## Listeners (`listeners.js`)

Registered at startup. Listens to bus events for:

- **Embedding** — `decision.created` triggers async embedding via `embedText()`, storing the vector in `_v2_dc_decisions.embedding`
- **Anamnesis** — `decision.created` queries the 5 most similar decisions by cosine distance (`<=>`) and optionally posts recommendations to the linked teamchat room
- **KAG auto-indexing** — `decision.created` extracts factual entities from title+description via LLM, writes them to `kag_entities` and `kag_relations` (source=`decision`, source_version=decisionId)
- **KAG re-indexing** — `decision.updated` deletes old KAG data for the decision, then re-extracts and re-imports
- **KAG cleanup** — `decision.deleted` removes all KAG entities and relations sourced from that decision

## AI Tools

| Tool | Risk Tier | Description |
|------|-----------|-------------|
| `search_similar_decisions` | TIER_LOW | Vector + FTS hybrid search across decisions |
| `create_decision` | TIER_MEDIUM | Create a new decision with title, domain, description, team, impact (critical\|high\|medium\|low). Optionally creates a chat room. |
| `update_decision` | TIER_MEDIUM | Update decision fields: title, domain, verdict, description, impact (critical\|high\|medium\|low) |
| `get_decision` | TIER_LOW | Get a single decision by ID with links and discussions |
| `list_decision_links` | TIER_LOW | List links for a decision grouped by rel_type |
| `analyze_decision_conflicts` | TIER_LOW | Analyze conflicts for a decision (direct, transitive, KAG, memory) |
| `delete_decision` | TIER_HIGH | Delete a decision (irreversible) |
| `create_decision_link` | TIER_MEDIUM | Create a typed link between two decisions |
| `delete_decision_link` | TIER_HIGH | Delete a link between decisions (irreversible) |
| `list_decisions` | TIER_LOW | List decisions with filters and pagination |
| `get_decision_history` | TIER_LOW | Get change history for a decision |
| `get_decision_iterations` | TIER_LOW | List Meta KB iterations linked to a decision |

## KAG Indexing

KAG indexing is handled by `kag-index.js` and triggered automatically on `decision.created` and `decision.updated` bus events.

- Extracts 10 entity types: Technology, Organization, Component, Concept, Standard, Domain, Person, Event, Project, Place
- Establishes 8 relation types: USES, REPLACES, CONFLICTS, PART_OF, DEPENDS_ON, RELATED_TO, IMPLEMENTS, COMPARED_TO
- Entity IDs are stable, generated via md5 hash (12 hex chars)
- On update, uses a delete-then-reimport pattern: all KAG entities and relations sourced from the decision are removed before re-extraction and re-import

## Events Emitted

- `decision.created` — `{ db, pool, decisionId, title, domain, verdict, createdBy, description }`
- `decision.updated` — `{ db, pool, decisionId }`
- `decision.deleted` — `{ db, pool, decisionId }`
- `decision.link.created` — `{ db, pool, decisionId, toId, relType, createdBy }`
