# Module: graph

**Path:** `src/api/v2/modules/graph/`
**Files:** `router.js`, `service.js`, `listeners.js`
**Base URL:** `/api/v2/:db/graph/...`
**Auth:** JWT required for all endpoints except `/graph/health`. Workspace access is judged solely by the shared `workspaceRoleMiddleware` mounted on `/:db` above the graph router — the module carries no membership guard of its own (`requireWorkspaceMembership` was removed in `c34e12b4a`). Non-members are still rejected with 403; superadmins pass as implicit `admin`, and membership through an organization counts. Write endpoints additionally require `requireRole('editor')`.

## Purpose

Knowledge graph layer on PostgreSQL. Every EAV object can be a graph node; relationships between objects are edges. Also hosts the agent memory REST API. Supports vector search via pgvector.

## Endpoints

### Graph nodes

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/graph/nodes/:objId` | viewer | Get node + neighborhood (`direction`, `limit`) |
| GET | `/graph/nodes?typeId=` | viewer | List nodes for a type (`includeEdges=true` for edges too) |
| POST | `/graph/nodes` | editor | Upsert a node manually |
| DELETE | `/graph/nodes/:objId` | editor | Delete a node |

### Graph edges

| Method | Path | Role | Description |
|--------|------|------|-------------|
| POST | `/graph/edges` | editor | Create/replace an edge (`fromObjId`, `toObjId`, `relType`) |
| DELETE | `/graph/edges` | editor | Delete an edge |

### Traversal

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/graph/path?from=&to=&maxHops=` | viewer (JWT) | Shortest path BFS (max 10 hops) |

### Vector search

| Method | Path | Role | Description |
|--------|------|------|-------------|
| POST | `/graph/vector-search` | viewer (JWT) | Semantic search over objects or doc chunks (`query`, `topK`, `type`) |

### Agent memory

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/graph/agents/:agentId/memory` | viewer | List agent memory entries (`?tag=`) |
| PUT | `/graph/agents/:agentId/memory/:key` | editor | Write memory entry |
| DELETE | `/graph/agents/:agentId/memory/:key` | editor | Delete memory entry |
| POST | `/graph/agents/:agentId/memory/:key/link` | editor | Link memory entry to an EAV object |
| GET | `/graph/memory/agents` | viewer | List agents with memory in this workspace |
| GET | `/graph/memory?agent_id=` | viewer | Full memory graph (nodes + SIMILAR_TO edges) |

### Health

| Method | Path | Description |
|--------|------|-------------|
| GET | `/graph/health` | Check PostgreSQL graph connectivity |

## Node Enrichment

When returning nodes, the API enriches `val` (display name) by querying the first non-empty CHARS(8)/SHORT(3)/HTML(2)/type 12 requisite value for the object. Falls back to the object's own `val`, then `#objId`.

## Listeners (`listeners.js`)

Subscribes to event bus:
- `object.edge.created` → `svc.upsertEdge()`
- `object.edge.deleted` → `svc.deleteEdge()`

EAV object CRUD → graph node sync is handled by `object.created/updated/deleted` listeners in `graph/listeners.js`.

## Micro-batching & Backfill

### Micro-batching (listeners.js)

Listeners buffer events and flush every 300ms or 100 events (whichever comes first) to reduce DB round-trips. Buffered operations: `upsertNode`, `deleteNode`, `upsertEdge`, `deleteEdge`. Node upserts are batched into multi-value `INSERT ... ON CONFLICT` grouped by workspace. Node deletes cascade to edges.

### Retry Queue

Failed flushes are pushed to a retry queue (max 10000 events). Retries use exponential backoff delays: 5s, 15s, 60s. After 3 attempts, events are dropped with a warning — the backfill system recovers.

### Backfill System (`backfillGraph()`)

Runs at startup in the background. For each workspace, checks if `graph_objects`/`graph_edges` are empty or have a pending checkpoint, then rebuilds:

- **Node backfill** — reads EAV instances (rows whose parent is a type definition), inserts in batches of 500 with `ON CONFLICT` upsert. Supports checkpointing via `_v2_backfill_state` table (columns: `job_id`, `last_obj_id`, `processed_count`, `last_checkpoint`) — resumes from last checkpoint after interruption.
- **Edge backfill** — finds REF-type column definitions, reads ref data rows, inserts edges in batches of 500 with `ON CONFLICT DO NOTHING`.
- Checkpoint is deleted after successful completion.

## Service Exports (`service.js`)

| Function | Description |
|----------|-------------|
| `upsertObjectNode(db, typeId, objId, val, extra)` | Create/update a graph node for an EAV object |
| `deleteObjectNode(db, objId)` | Delete a node and all its edges |
| `getObjectNode(db, objId)` | Get a single node by db+objId |
| `listObjectNodes(db, typeId, opts)` | List nodes for a type (limit, skip) |
| `listObjectNodesWithEdges(db, typeId, opts)` | List nodes + edges between them |
| `upsertEdge(db, fromObjId, toObjId, relType, props)` | Create/replace a directed edge |
| `deleteEdge(db, fromObjId, toObjId, relType)` | Delete an edge by endpoints + type |
| `getObjectNeighborhood(db, objId, opts)` | Get 1-hop neighborhood (direction, limit) |
| `shortestPath(db, fromObjId, toObjId, maxHops)` | BFS shortest path (recursive CTE, default 6, max 10 hops enforced by router) |
| `deleteDocChunks(db, docId)` | Delete all doc chunks for a document |
| `backfillGraph()` | Full graph rebuild from EAV for all workspaces |

## Notes

- `vectorSearchObjects` and `vectorSearchDocs` used by the `/graph/vector-search` endpoint are imported from `backend/src/api/v2/utils/embedding-sync.js` (not `service.js`). They accept plain text, compute the embedding internally, then run pgvector similarity search. The low-level variants in `service.js` accept a pre-computed vector and are used internally by the backfill pipeline.
- `graph_query` tool (MCP/tool-executor): the TOOL_DEF schema declares only `cypher` as the parameter name, but the tool-executor also accepts `query` as a fallback alias (`args.cypher || args.query`). Only `cypher` should be used in new code.

## KAG (Knowledge-Augmented Graph)

**Path:** `src/api/v2/modules/swarm-memory/kag.js`
**Base URL:** `/api/v2/:db/swarm-memory/kag/...`
**Auth:** JWT required. Admin-only for import/delete/sync operations.

KAG is a structured knowledge graph layer on top of PostgreSQL, separate from the EAV object graph. It stores typed entities, relations, and ontology classes with vector embeddings for hybrid search. Used by AI agents via `kag_search`, `kag_traverse`, `kag_ask` shared tools.

### KAG Endpoints

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/kag/search` | viewer | Hybrid vector+FTS search. Params: `q` (required), `limit`, `offset`, `tags` (comma-separated). First page (offset=0) returns vector+FTS results and `total`; subsequent pages use FTS-only pagination. |
| POST | `/kag/edges` | viewer | Bulk fetch relations for entity IDs. Body: `{ ids, tags }`. Returns `{ edges, neighbors }` — neighbors are entities referenced by edges but not in the input set. |
| GET | `/kag/traverse` | viewer | Graph traversal from an entity. Params: `entityId` (required), `depth` (default 2), `relType`, `tags` (comma-separated). Max 50 results. |
| GET | `/kag/stats` | admin | Entity/relation/class counts for the workspace. |
| POST | `/kag/import-ontology` | admin | Import ontology classes. Body: `{ classes: [{ id, name, description?, parentClassId? }], source, version }`. |
| POST | `/kag/import-entities` | admin | Import entities. Body: `{ entities: [{ id, name, entityType?, observations?, properties?, classId?, tags? }], source, version, tags? }`. Top-level `tags` applied to all entities. |
| POST | `/kag/import-relations` | admin | Import relations. Body: `{ relations: [{ id?, sourceId, targetId, type, properties? }], source, version }`. |
| DELETE | `/kag/data` | admin | Clear KAG data. Optional `?source=` filter. |
| POST | `/kag/sync-memory-files` | admin | Sync memory files into KAG. Body: `{ memoryDir }`. Requires `KAG_MEMORY_DIRS` env allowlist. |
| PATCH | `/kag/entities/:entityId/tags` | admin | Update entity tags. Body: `{ tags: string[] }`. |
| GET | `/kag/clusters` | viewer | Get entity clusters (community detection). |
| GET | `/kag/anomalies` | viewer | Detect graph anomalies (isolated nodes, unusual patterns). |
| GET | `/kag/browse` | viewer | Generic KAG entity browser. Params: `entityType`, `source`, `limit`, `offset`, plus any extra query params passed as `properties` filter. |

### Tags System

Entities have a `tags TEXT[]` column with a GIN index. Tags enable access-scoped filtering: the workspace agent config field `kagTags` is passed through tool context, so each agent query automatically filters to its permitted tag set. All search/traverse/edges endpoints accept a `tags` parameter to restrict results. Entities with no tags or empty tags are visible to all scopes.

## AI Tools (graph group)

| Tool | Risk Tier | Description |
|------|-----------|-------------|
| `get_related` | LOW | Find related objects via graph traversal. Params: `objId` (required), `relType`, `depth` (1-3, default 2). |
| `get_graph_node` | LOW | Get a graph node by object ID. Returns type, name, edges. Params: `objId` (required). |
| `get_graph_neighborhood` | LOW | Get 1-hop neighbors of a node. Params: `objId` (required), `direction` (in/out/both, default both), `limit` (default 100). |
| `list_graph_nodes` | LOW | List graph nodes by table/type ID. Params: `typeId` (required), `withEdges` (bool), `limit`, `skip`. |
| `get_shortest_path` | LOW | BFS shortest path between two objects. Params: `fromObjId`, `toObjId` (required), `maxHops` (default 6). |
| `upsert_graph_node` | MEDIUM | Create or update a graph node. Params: `objId`, `typeId` (required), `val`, `parentId`. |
| `delete_graph_node` | HIGH | Delete a node and all its edges. Irreversible. Params: `objId` (required). |
| `upsert_graph_edge` | MEDIUM | Create or update a directed edge. Params: `fromObjId`, `toObjId`, `relType` (required), `properties` (JSONB). |
| `delete_graph_edge` | HIGH | Delete an edge. Irreversible. Params: `fromObjId`, `toObjId`, `relType` (required). |

## DB Tables (global schema)

- `graph_objects` — `obj_id`, `db`, `type_id`, `val`, `parent_id`, `updated_by`, `updated_at`, `node_type TEXT NOT NULL DEFAULT 'object'`, `meta JSONB`, `embedding` (vector(1024)), `embedding_hash`, HNSW index
- `graph_edges` — `from_obj_id`, `to_obj_id`, `rel_type`, `db`, `properties` (JSONB)
- `doc_chunks` — `doc_id`, `chunk_idx`, `text`, `embedding` (vector(1024)), `db`
- `agent_memory` — `key`, `agent_id`, `db`, `value` (JSONB), `tags`, `embedding` (vector(1024)), `tsvector`
- `kag_entities` — `db`, `id` (PK: db+id), `name`, `entity_type`, `observations TEXT[]`, `properties JSONB`, `embedding` (vector(1024)), `source`, `source_version`, `status VARCHAR(16)`, `tags TEXT[]` (GIN index), `fts tsvector` (GIN index), `created_at`, `updated_at`. HNSW index on embedding.
- `kag_classes` — `db`, `id` (PK: db+id), `name`, `description`, `parent_id`, `source`, `source_version`, `created_at`, `updated_at`
- `kag_relations` — `db`, `source_id`, `target_id`, `rel_type` (PK: db+source_id+target_id+rel_type), `properties JSONB`, `source`, `source_version`. Index on `(db, target_id)`.
