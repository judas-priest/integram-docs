# Nightcall Module

Specification-driven AI execution — the Nightcall module implements a managed synthesis and verification pipeline (Canonical Normative Model → EffectiveSpecification → ExecutionBundle → Evidence → Decisions).

## Architecture

Systems A–F from the Nightcall architecture:

| System | File | Role |
|---|---|---|
| A | `proposals.js` + `llm.js` + `ncl_formalize` | Extraction IR → ModelChangeProposal → GovernanceDecision |
| B | `service.js`, `artifact-specs.js` | CNM storage: requirements, intents, sources, edges, decisions, artifact specs |
| C | `resolver.js` | EffectiveSpecification resolver (strict applicability, exceptions, priorities) |
| D | `compiler.js` | Execution compiler (D1 deterministic handlers + D2 mapping proposals) |
| E | `engine.js` + `dag.js` | DAG-based generation with directed retry and diagnosis |
| F | `evidence.js` | Verification runners, EvidenceEnvelope, Compliance/Release decisions, waivers |

## Tables (11)

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
- `GET /conflicts/:requirementId`
- `POST /decisions`
- `POST|GET /proposals`, `GET /proposals/:id`, `POST /proposals/:id/decide`
- `POST|GET /artifact-specs`
- `POST /resolve`, `POST /run`

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

Known limitation (authority model, TD-NCL-13): the engine reads workspace tables as the
run executor with no per-table grant check; access control is enforced at the REST/tool
boundary.

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
This is the role half of the authority model; computable `Authority` objects from the
canon (02 §9) remain open — see TD-NCL-13.

Waivers accept an optional `expires_at`; an expired waiver stops suppressing its
violation and the release verdict returns to `REJECTED`.

## Integration Points

- **Integram → Nightcall**: EAV data as source facts, documents as source artifacts, automations for invalidation
- **Nightcall → Integram**: Block documents as ArtifactView, notifications on run completion
