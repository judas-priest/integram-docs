# Nightcall Module

Specification-driven AI execution — the Nightcall module implements a managed synthesis and verification pipeline (Canonical Normative Model → EffectiveSpecification → ExecutionBundle → Evidence → Decisions).

## Architecture

Systems A–F from the Nightcall architecture:

| System | File | Role |
|---|---|---|
| A | `proposals.js` + `llm.js` + `ncl_formalize` | Extraction IR → ModelChangeProposal → GovernanceDecision |
| B | `service.js`, `artifact-specs.js`, `families.js`, `baselines.js` | CNM storage: requirements, intents, sources, edges, decisions, artifact specs, families, baselines |
| C | `resolver.js` | EffectiveSpecification resolver (strict applicability, exceptions, priorities) |
| D | `compiler.js`, `methods.js` | Execution compiler (D1 deterministic handlers + D2 mapping proposals), verification method registry |
| E | `engine.js` + `dag.js` + `unit-states.js` | DAG-based generation with directed retry, diagnosis, and persisted unit states |
| F | `evidence.js`, `consilium.js` | Verification runners, EvidenceEnvelope, Compliance/Release decisions, multi-expert deliberation |
| — | `authority.js` | Scoped authority grants and waivers (canon 02 §9) |
| — | `outbox.js` | Domain event bus (18 event types, at-least-once delivery) |
| — | `source-plane.js` | Source plane connectors, cursor advancement, collection tracking |
| — | `listeners.js` | EAV change → STALE cascade for dependent evidence |

## Tables (25)

| Table | Purpose |
|---|---|
| `_v2_ncl_sources` | SourceStatement fragments from documents (audited updates) |
| `_v2_ncl_requirements` | Versioned requirements in CNM |
| `_v2_ncl_intents` | VerificationIntents (what to verify) |
| `_v2_ncl_edges` | Typed graph edges (derivedFrom, supersedes, exceptionTo, conflictsWith...) |
| `_v2_ncl_proposals` | ModelChangeProposals holding Extraction IR (System A governance path) |
| `_v2_ncl_artifact_specs` | Versioned ArtifactSpecifications (schema nodes + generation edges) |
| `_v2_ncl_runs` | Execution runs with frozen spec and InputSnapshot |
| `_v2_ncl_attempts` | Generation attempts within a run (each retry carries the previous attempt's diagnosis) |
| `_v2_ncl_evidence` | Immutable EvidenceEnvelopes |
| `_v2_ncl_decisions` | Governance, Compliance, Release, Waiver decisions |
| `_v2_ncl_obligations` | Per-run verification obligations |
| `_v2_ncl_methods` | Verification method registry (deterministic `execute` + `explain`) |
| `_v2_ncl_source_definitions` | Source plane connector definitions (type, config, cursor) |
| `_v2_ncl_collection_jobs` | Source plane collection job runs |
| `_v2_ncl_collection_attempts` | Per-source collection attempts with error tracking |
| `_v2_ncl_collection_incidents` | Collection incidents (failures, anomalies) |
| `_v2_ncl_raw_captures` | Manifest/reference of collected source data |
| `_v2_ncl_normalized_records` | Quality and freshness assessment of captured data |
| `_v2_ncl_outbox` | Domain event outbox (18 event types, at-least-once delivery) |
| `_v2_ncl_authority_grants` | Scoped authority grants and waivers |
| `_v2_ncl_families` | Artifact families — runs grouped by `artifact_type` |
| `_v2_ncl_baselines` | Frozen requirement baselines for comparison |
| `_v2_ncl_baseline_items` | Individual items within a baseline snapshot |
| `_v2_ncl_verification_runs` | Per-obligation verification execution audit trail |
| `_v2_ncl_unit_states` | Per-unit state transitions persisted during engine execution |

## AI Tools (21)

| Tool | Tier | Description |
|---|---|---|
| `ncl_create_requirement` | MEDIUM | Create requirement (draft/candidate only; higher statuses require governance) |
| `ncl_list_requirements` | LOW | List/search requirements |
| `ncl_get_requirement` | LOW | Get requirement with graph |
| `ncl_update_requirement` | MEDIUM | Update requirement (status machine enforced) |
| `ncl_create_edge` | MEDIUM | Create typed graph edge |
| `ncl_get_graph` | LOW | Get subgraph around node |
| `ncl_create_intent` | MEDIUM | Create VerificationIntent |
| `ncl_find_conflicts` | LOW | Find conflicting requirements |
| `ncl_governance_decision` | HIGH | Accept/reject a single requirement |
| `ncl_create_source` | MEDIUM | Create SourceStatement |
| `ncl_update_source` | MEDIUM | Update SourceStatement (cascades STALE to derived requirements' evidence) |
| `ncl_formalize` | MEDIUM | System A: text → Extraction IR → ModelChangeProposal |
| `ncl_list_proposals` | LOW | List ModelChangeProposals |
| `ncl_get_proposal` | LOW | Get proposal with Extraction IR |
| `ncl_decide_proposal` | HIGH | Governance decision; acceptance publishes IR into CNM |
| `ncl_save_artifact_spec` | HIGH | Save new ArtifactSpecification version |
| `ncl_list_artifact_specs` | LOW | List artifact specifications |
| `ncl_propose_mapping` | MEDIUM | D2: propose backend mapping for unknown intent kind (candidate) |
| `ncl_waive` | HIGH | Authorized waiver of a known violation |
| `ncl_resolve_spec` | MEDIUM | Resolve EffectiveSpecification |
| `ncl_run` | HIGH | Full pipeline: resolve → compile → generate → verify → decide (with directed retry) |

## REST Endpoints

All under `/:db/nightcall/` (requireJwt + workspaceRoleMiddleware):

- `GET /health`
- `POST|GET /requirements`, `GET|PATCH /requirements/:id`, `POST /requirements/:stableId/versions`
- `POST|GET /edges`, `GET /graph/:nodeType/:nodeId`
- `POST|GET /intents`
- `POST /sources`, `PATCH /sources/:id`
- `GET /sources/definitions`, `GET /sources/definitions/:id`
- `GET /conflicts/:requirementId`
- `POST /decisions`
- `POST|GET /proposals`, `GET /proposals/:id`, `POST /proposals/:id/decide`
- `POST|GET /artifact-specs`
- `POST /resolve`, `POST /run`
- `GET|POST /runs`, `GET /runs/:id`, `POST /runs/:id/view`
- `GET /compare/:runA/:runB` — see "Comparison" below
- `GET|POST /baselines`, `GET /baselines/:id`
- `GET /families`, `GET /families/:id`
- `GET /outbox`, `POST /outbox/drain`
- `GET /authority/grants`, `POST /authority/grants`
- `GET /authority/waivers`, `POST /authority/waivers`


### Comparison

`GET /compare/:runA/:runB` returns the seven slices of canon 12 §10.4. The seventh —
business dimensions — comes from the `comparison_policy.dimensions` of the family the
runs belong to; run A's family wins, run B's is the fallback. The response carries
`businessDimensionsSource` alongside the list (`caller`, `family:<id>`,
`family:<id>:unavailable`, `none`) so an empty list is never mistaken for "no business
differences".

Declared structural operations (SPLIT / MERGE / REDEFINE) are NOT wired into the
comparison: `computeSchemaDelta` can only infer REMOVE, MOVE, RETYPE, RENAME, PRESERVE,
ADD and CHANGE_DEPENDENCY on its own, so the `SPLIT` and `MERGED` outcomes of the
migration plan are unreachable today. See TD-NCL-33.

## Key Concepts

- **Requirement**: normative statement with modality, subject/predicate/object, status lifecycle (`draft → candidate → [needs_clarification] → proposed → accepted → active → contested/superseded/retired`). Statuses beyond candidate require a GovernanceDecision — on both create and update paths.
- **ModelChangeProposal**: the ONLY path from a source into the CNM. `ncl_formalize` stores the Extraction IR here; `ncl_decide_proposal` publishes it (statuses: proposed, deciding, accepted, rejected, needs_clarification, apply_failed). Unresolvable inter-requirement edges are surfaced as `skippedEdges`, never silently dropped.
- **EffectiveSpecification**: immutable context slice. Applicability is three-valued: applicable / not applicable / **unresolved** (missing TaskContext keys or malformed activation conditions block preflight with `BLOCKED_MISSING_CONTEXT`). Exceptions (`exceptionTo`) and priority-based conflict resolution are applied with a decision trace; equal-priority conflicts block; intent kinds without a registered handler block preflight (`NO_HANDLER`).
- **ArtifactSpecification**: versioned artifact type schema consumed by the compiler; built-in QBR/report schemas are the fallback. Saving inserts the new active version before superseding old ones (no zero-active window).
- **InputSnapshot**: frozen into every run before generation — task context hash (key-order independent), requirement versions, artifact spec version, explicit `reproducibilityLimit`.
- **Directed retry**: REJECTED runs retry within budget; each attempt's row carries the PREVIOUS attempt's diagnosis (what directed this retry); repeated candidate hashes stop the loop ("no progress", oscillation-proof via seen-hash set). Compilation coverage gate: uncovered obligations block execution entirely.
- **EvidenceEnvelope**: typed verification result — `relation` (SUPPORTS/REFUTES/INCONCLUSIVE) kept separate from `run_status` and `freshness`.
- **Decisions**: governance / compliance (per obligation, linked to run; only the LATEST decision per obligation counts toward release) / release (`ACCEPTED | REJECTED | WAIVED | HUMAN_REVIEW | ABORTED`) / waiver (requires rationale; standing waivers are requirement-wide).
- **STALE ≠ FAIL**: meaningful requirement changes (status, text, effective window, activation condition) and any source change mark dependent evidence STALE, preserving the original verdict.

## Generation (System E)

`generation.js` gathers workspace EAV data once per run (`getAllTypes` + `listObjects`,
capped at 10 tables × 50 rows, optional `dataSources` table-name allowlist) and freezes
a content-hashed manifest into the run's InputSnapshot. Each GenerationUnit is then
generated by `callLLMJson` with: the section's schema node, upstream node values,
per-node requirements (from `artifactSchema.requirementCoverage`), and the data manifest.
Modes: `ai` (default) and `builtin` (deterministic fixture in `builtin-content.js`, also
the per-node fallback when the LLM fails — reported as `generationMode: 'ai_with_fallback'`).
The engine loads `generation.js` lazily inside `executeFullRun` so importing `engine.js`
never drags in the llm/ai module chains.

Directed retry regenerates only the diagnosed subgraph: failed obligation → schema node
(via verificationPlan fieldPath, falling back to requirementCoverage) → `gen_<node>` unit
+ `getDownstream`. Unmappable failures regenerate everything. All obligations are
re-verified every attempt (regression check).

Structural fieldPaths are resolved at compile time against real schema nodes
(normalized + squashed matching); a target that maps to no node compiles with
`unmapped: true` and verifies as INCONCLUSIVE (mapping gap), never REFUTES.

EAV changes cascade STALE (listeners.js): a changed object that is a registered
SourceArtifact invalidates evidence of requirements derivedFrom it; a changed table
present in a run's InputSnapshot manifest invalidates that run's evidence
(`to_regclass` guard keeps non-nightcall workspaces untouched, no DDL).

Note: the engine reads workspace tables as the run executor with no per-table grant check;
access control is enforced at the REST/tool boundary.

## Access control

The module is opt-in: `settings.modules.nightcall` must be explicitly `true`
(`PUT /api/v2/workspaces/:slug` with `{"settings":{"modules":{"nightcall":true}}}`).
Two independent gates enforce it:

- **REST** — `requireModule('nightcall', { defaultEnabled: false })` on the router mount.
- **Tools** — `ai/agent/module-gate.js` hides every `ncl_*` tool from `GET /ai/tools`
  and refuses it in `permissionMiddleware`, so neither MCP nor the in-app agent can
  call it in a workspace that has not enabled the module. Unknown workspace settings
  fail closed. The permission check runs before the HITL check, so a refused tool
  never writes a pending confirmation.

Authority (identical through both doors):

| Operation | Role |
|---|---|
| Reads (`GET`) | any member |
| Writes, `POST /resolve`, `POST /run` | `editor` |
| `POST /proposals/:id/decide`, `POST /decisions`, `ncl_decide_proposal`, `ncl_governance_decision`, `ncl_waive` | `admin` |

Mutating REST routes also run `csrfMiddleware` (Bearer/service-key requests are exempt).

### Authority Model (`authority.js`)

Beyond workspace roles, the module implements a computable authority model (canon 02 §9)
with seven named authorities: `APPROVE_INTERPRETATION`, `DEFINE_TERM`, `GRANT_EXCEPTION`,
`GRANT_WAIVER`, `ACCEPT_SEMANTIC_RISK`, `CHANGE_RELEASE_POLICY`, `DECLASSIFY`.

`ROLE_BASELINE` is derived from the platform's `ROLE_HIERARCHY`, not listed by hand:
every role at or above `admin` (i.e. `admin` and `owner`) gets all authorities by
default, everything below gets none unless explicitly granted. It used to be a
hand-written table and drifted — the platform added `owner`, the module did not, and the
workspace owner was locked out of governance actions a plain admin could perform. See
TD-NCL-31.

Grants are stored in `_v2_ncl_authority_grants` with optional scope (JSON object —
empty = global, otherwise all keys must match) and expiration. `hasAuthority()` checks
role baseline first, then scoped grants.

Waivers are a specialized grant (`authority = 'GRANT_WAIVER'`, `subject = 'obligation:<id>'`).
`createWaiver` emits a `GovernanceDecisionMade` event through the outbox.

**These grants do not affect a release verdict.** `makeReleaseDecision` suppresses a
violation only from a *decision* row (`decision_type = 'waiver'` in `_v2_ncl_decisions`,
written by `createDecision`); it never reads `_v2_ncl_authority_grants`. The expiry rule
— an expired waiver stops suppressing its violation and the verdict returns to
`REJECTED` — applies to those decision rows. Reconciling the two stores is open work;
see TD-NCL-33's neighbours in `docs/tech-debt-nightcall.md`.

## Outbox (`outbox.js`, `outbox-worker.js`)

Domain event bus with at-least-once delivery. 18 event types covering the full lifecycle.

**Transactional guarantee:** `withOutbox(pool, db, eventType, payload, domainFn)` wraps
the domain write and event INSERT in a single `BEGIN/COMMIT` transaction. All callers in
service.js, proposals.js, evidence.js, authority.js, artifact-specs.js, and engine.js use
this helper. `baselines.js` passes its existing `client` with `inTransaction: true`.

**Background drain:** `outbox-worker.js` registers a BullMQ repeatable job (`nightcall-outbox`
queue) that drains unpublished events every 30 s. Started lazily on the first **REST**
request to the nightcall router. AI/MCP tool calls do not reach that router, so a
workspace driven only through the agent never starts the drain and accumulates rows; see
TD-NCL-34. Falls back to manual drain (`POST /outbox/drain`) when Redis is unavailable.

Events are published to the event bus as `ncl.<EventType>`. After `MAX_ATTEMPTS=5` failures
the row remains visible as permanent debt, never deleted.

## Source Plane (`source-plane.js`)

Connector subsystem for ingesting external source data. Source definitions
(`_v2_ncl_source_definitions`) describe a source type, config, and cursor position.
`getSourceDefinition` retrieves a definition; cursor advancement tracks incremental
collection progress. Collection jobs, attempts, and incidents are tracked in their
respective tables.

## Verification Methods (`methods.js`)

Registry of verification methods. Each method implements an `execute(obligation, artifact)`
function returning an EvidenceEnvelope and an `explain()` function returning a human-readable
description. Five deterministic verifiers are ported as `execute` functions. Methods are
stored in `_v2_ncl_methods` and looked up by intent kind during compilation.

## Unit States (`unit-states.js`)

Persists per-unit state transitions during engine execution. `setUnitState(pool, db, runId,
unitId, state, meta)` writes to `_v2_ncl_unit_states`; `getUnitStates(pool, db, runId)`
retrieves the full state history for a run. The engine writes state transitions
(`pending → generating → verifying → done/failed`) as units progress through the DAG.

**Gate enforcement:** On retry attempts (attempt > 1), the engine calls `evaluateGate`
before generating each unit. Blocked units receive state `BLOCKED` with the specific
reason (`missing_evidence`, `stale_evidence`, `wrong_payload_type`, `assurance_too_low`)
and their previous value is reused. First attempts run all units unconditionally
(no prior evidence exists).

Gate policy is derived per unit from the intent-level declarations that reach the
obligation (`required_assurance`, `evidence_policy`): the strictest assurance wins and
accepted payload types are unioned. A blocked unit still gets its obligation verified,
so a truncated artifact cannot reach `ACCEPTED` silently — the obligation lands in
`undecided` and the run returns `HUMAN_REVIEW`.

The first attempt is open by design: verification runs over the whole artifact, not per
unit. Closing that half needs per-unit verification — see TD-NCL-30.

## Consilium (`consilium.js`)

Multi-expert deliberation for verification decisions. Calls LLM with retry on empty
responses (token budget 1024, smart truncation of obligation context). Used when a
verification obligation requires judgment beyond deterministic methods.

## Integration Points

- **Integram → Nightcall**: EAV data as source facts, documents as source artifacts, automations for invalidation
- **Nightcall → Integram**: Block documents as ArtifactView, notifications on run completion
