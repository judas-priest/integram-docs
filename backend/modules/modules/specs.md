# Module: specs

Declarative data quality specs — invariant checks for EAV records.

## Data Specs

**Path:** `src/api/v2/modules/specs/`
**Files:** `router.js`, `service.js`, `config.js`, `evaluator.js`
**Base URL:** `/api/v2/:db/specs/...`
**Auth:** JWT required. CRUD: `admin`. Check/run: any authenticated role.

### Purpose

Declarative data invariants (ADR-019). A spec is a named set of rules bound to a table type; the evaluator checks EAV records against those rules without DB or LLM calls.

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/specs/types/:typeId` | List specs for a table |
| POST | `/specs/types/:typeId` | Create spec (admin) |
| PATCH | `/specs/:specId` | Update spec (admin) |
| DELETE | `/specs/:specId` | Delete spec (admin) |
| POST | `/specs/:specId/run?limit=` | Run spec across all records of its table |
| GET | `/specs/types/:typeId/records/:objectId/check?specId=` | Check one record against spec(s) |

### Data Model

**Table:** `_v2_data_specs` (per-workspace, lazy-init)

| Column | Type | Description |
|--------|------|-------------|
| `id` | BIGINT (identity) | PK |
| `type_id` | BIGINT | EAV type this spec belongs to |
| `name` | VARCHAR(255) | Human-readable name |
| `definition` | JSONB | `{ rules: [{ field, op, value?, validator? }] }` |
| `enabled` | BOOLEAN | Default `true` |
| `created_at` | TIMESTAMPTZ | |
| `created_by` | TEXT | |

### Evaluator

Pure function (`evaluator.js`). Rules are combined with AND; a spec passes only when all rules pass.

| Operator | Description |
|----------|-------------|
| `required` | Field must be non-empty |
| `notEmpty` | Alias for `required` |
| `valid` | Field must pass a validator (`phone`, `email`, `text`); reuses resolution validators |
| `equals` | Field must equal `rule.value` |
| `notEquals` | Field must not equal `rule.value` |
| `in` | Field must be in a comma-separated list or array `rule.value` |
| `matches` | Field must match regex `rule.value`; ReDoS protection rejects nested quantifiers |
| `min` | Numeric field must be >= `rule.value` |
| `max` | Numeric field must be <= `rule.value` |

---

## AI Tools

| Tool | Risk Tier | Description |
|------|-----------|-------------|
| `list_specs` | TIER_LOW | List data specs for a table |
| `check_record` | TIER_LOW | Check a record against its specs |
| `run_spec` | TIER_LOW | Run data spec across all records of its table |

## References

- ADR-019 (declarative data specs)
