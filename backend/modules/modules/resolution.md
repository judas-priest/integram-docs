# Module: resolution

**Path:** `src/api/v2/modules/resolution/`
**Files:** `router.js`, `service.js`, `config.js`, `survivorship.js`, `match.js`, `validators.js`, `listeners.js`
**Base URL:** `/api/v2/:db/resolution/...`
**Auth:** JWT required. Config CRUD: `admin`. Recompute: `editor`. Verify/lineage: any authenticated role.

## Purpose

Golden record synthesis for client identity resolution (ADR-018). Given multiple source rows for the same client (e.g. from different systems), deterministically computes a single golden value per field using configured source priorities and validators. No LLM in the merge path.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/resolution/:typeId/config` | Get resolution config for a table |
| PUT | `/resolution/:typeId/config` | Save/upsert resolution config (admin) |
| DELETE | `/resolution/:typeId/config` | Delete resolution config (admin) |
| POST | `/resolution/:typeId/clients/:clientId/recompute` | Recompute golden record (editor) |
| GET | `/resolution/:typeId/clients/:clientId/verify` | Shipping-readiness check (read-only) |
| GET | `/resolution/:typeId/clients/:clientId/lineage` | Latest lineage per field |

## Data Model

**Table:** `_v2_resolution_config` (per-workspace, lazy-init)

| Column | Type | Description |
|--------|------|-------------|
| `type_id` | BIGINT | PK, client table type ID |
| `config` | JSONB | Resolution rules (see Config Shape below) |
| `updated_at` | TIMESTAMPTZ | |
| `updated_by` | TEXT | |

**Table:** `_v2_resolution_lineage` (per-workspace, lazy-init)

| Column | Type | Description |
|--------|------|-------------|
| `id` | BIGINT (identity) | PK |
| `object_id` | BIGINT | Client record ID |
| `field_key` | TEXT | Golden field requisite ID |
| `value` | TEXT | Resolved value |
| `source` | TEXT | Winning source identifier |
| `source_row_id` | BIGINT | EAV ID of winning source row |
| `resolved_at` | TIMESTAMPTZ | |
| `resolved_by` | TEXT | |

### Config Shape (JSONB)

```json
{
  "sourceChildTypeId": 123,
  "sourceField": 45,
  "matchKeys": [{ "field": 10, "kind": "uds" }, { "field": 11, "kind": "phone" }],
  "fields": {
    "<goldenReqId>": {
      "sourceField": 67,
      "priority": ["source_a", "source_b"],
      "validator": "phone"
    }
  },
  "address": {
    "childTypeId": 314,
    "sourceField": 50,
    "goldenFlag": 51,
    "priority": ["source_a"]
  },
  "requiredForShipping": ["<goldenReqId>", "address"]
}
```

## Key Service Functions

- `recomputeClient(pool, db, typeId, clientId, user)` — load source rows, run survivorship merge, governed writeback of golden fields, flag golden address, append lineage. Emits `resolution.recomputed`.
- `verifyShipping(pool, db, typeId, clientId, user)` — read-only shipping-readiness check: required fields, valid values, no unresolved conflicts.
- `recomputeById(pool, db, clientId, user)` — MCP entry point, derives typeId.
- `verifyById(pool, db, clientId, user)` — MCP entry point, derives typeId.

## Survivorship Merge (`survivorship.js`)

Pure function. For each golden field: collects valid normalized candidates from all source rows, sorts by configured source priority then sourceRowId, picks the winner. Reports conflicts when multiple distinct normalized values exist.

`pickGoldenAddress` selects the winning address row by source priority.

## Strong-Key Matching (`match.js`)

`findClientByKeys(pool, db, clientTypeId, candidate, config)` — deterministic identity resolution at ingest. Tries configured match keys in order:

| Kind | Matching |
|------|----------|
| `uds` | Opaque exact match (trimmed) |
| `telegram` | Opaque exact match (trimmed) |
| `phone` | Normalized to E.164 (`+7XXXXXXXXXX`) then compared |
| `email` | Normalized (trim + lowercase) then compared |

No LLM — fuzzy name-only matching is a separate HITL-gated path (ADR-018).

## Validators (`validators.js`)

Pure normalizers returning canonical string or `null`:

- `normalizePhone(raw)` — Russian phone to E.164
- `normalizeEmail(raw)` — trim + lowercase
- `normalizeText(raw)` — collapse whitespace
- `validateField(kind, raw)` — dispatcher
- `matchKeyValue(kind, raw)` — dispatcher for strong-key matching

## Bus Listeners (`listeners.js`)

- `object.created` / `object.updated` — when a source-child or address-child row changes, auto-recomputes the parent client's golden record via `recomputeClient`. Golden writeback uses raw EAV writes (no bus events) to avoid feedback loops.

## AI Tools

| Tool | Risk Tier | Description |
|------|-----------|-------------|
| `resolve_client` | TIER_MEDIUM | Recompute golden record from source rows |
| `verify_client_shipping` | TIER_LOW | Shipping-readiness check (read-only) |

## References

- ADR-018 (source resolution)
