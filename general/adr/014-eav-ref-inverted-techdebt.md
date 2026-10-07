# ADR-014: EAV ref storage — inverted pattern techdebt

## Status: Accepted

## Context

EAV ref fields (status, payment, source — any field pointing to a lookup record) are stored in two formats:

- **Direct:** `t=colDefId, val=refObjId` — "field 378 has value 395"
- **Inverted:** `t=refObjId, val=colDefId` — "ref 395 belongs to field 378"

Both formats coexist in production data. When both exist for the same field on the same record, the record appears in multiple kanban columns (phantom duplicates).

## Current state (2026-06-02)

| Writer | Format | Cleanup |
|--------|--------|---------|
| UI (saveRequisites) | Inverted | Cleans direct |
| Telegram bot (upsertField/casSetField) | Inverted | Cleans direct |
| Connector (connectors/service.js) | **Inverted** (fixed 2026-06-02, was direct) | Cleans direct |
| Reader (loadAllRequisites, listObjects) | Reads both, prefers inverted | — |

All writers now use inverted pattern and clean up direct rows.

## Decision

Standardize on **inverted pattern** as the canonical format. This was the pragmatic choice — UI and bot (the most active writers) already use it.

## Techdebt

Inverted pattern is unintuitive: `t=395, val=378` requires knowing the EAV convention to understand. Direct pattern `t=378, val=395` reads naturally as "field=value".

### Future migration path (if pursued):
1. Switch all writers to direct: saveRequisites, upsertField, casSetField, connector
2. Migrate all existing inverted rows to direct
3. Simplify readers to only check direct
4. Update appendReqFilters to only use direct pattern
5. Update all tests

### Estimated scope:
- ~5 files to change writers
- ~70k EAV rows to migrate per workspace
- ~3 reader functions to simplify
- Risk: any missed writer creates inverted rows → duplicates return

### Recommendation:
Leave as-is until a concrete problem arises. Inverted works, all writers are aligned, cleanup prevents duplicates. The complexity is contained.
