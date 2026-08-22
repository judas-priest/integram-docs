# Module: reports

**Path:** `src/api/v2/modules/reports/`
**Files:** `router.js`, `service.js`, `schema.js`, `report-audit.js`
**Base URL:** `/api/v2/:db/reports/...`
**Auth:** JWT required for all endpoints.

## Purpose

SQL-backed report builder. A report is a named query over EAV tables with configurable columns (including aggregation), filters, joins, and sorting. Supports CSV export and SSE/NDJSON streaming for large datasets.

## Endpoints

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/reports` | any | List reports (`limit`, `offset`, `include`, `sort`, `q`) |
| POST | `/reports` | editor | Create report (idempotency key supported) |
| GET | `/reports/:reportId` | any | Get report structure (columns, metadata) |
| PATCH | `/reports/:reportId` | editor | Update report name, filters, WHERE clause |
| DELETE | `/reports/:reportId` | editor | Delete report |
| POST | `/reports/:reportId/run` | any | Execute report (JSON response with pagination) |
| GET | `/reports/:reportId/export` | any | Streaming CSV download |
| GET | `/reports/:reportId/stream` | any | SSE or NDJSON streaming for large reports |
| POST | `/reports/:reportId/columns` | editor | Add column to report |
| POST | `/reports/:reportId/columns/reorder` | editor | Reorder columns |
| PATCH | `/reports/:reportId/columns/:colId` | editor | Update column (alias, func, totalFunc, filters) |
| DELETE | `/reports/:reportId/columns/:colId` | editor | Delete column |
| POST | `/reports/:reportId/joins` | editor | Add a JOIN to another table |
| DELETE | `/reports/:reportId/joins/:joinId` | editor | Delete a JOIN |
| POST | `/reports/:reportId/bulk-update` | editor | Bulk update EAV objects via SET expressions |
| PATCH | `/reports/:reportId/visibility` | editor | Toggle report visibility (`hidden: true/false`). |
| GET | `/reports/:reportId/history` | any | Report audit history |

## Hidden Flag

Reports support a `hidden` boolean flag stored in `_v2_entity_meta` (key `hidden`, entity type `report`). Set via `PATCH /:reportId/visibility` with body `{ hidden: true/false }`. Hidden reports are excluded from the sidebar but remain accessible by direct ID. The flag is read by `listReports` and filtered client-side in the sidebar store.

## Formula Validation

Report columns support SQL formulas (e.g. `ROUND([THIS] * 1.2)`, `COALESCE([THIS], 0)`).

**Validation is at write-time only** (`addReportColumn`, `updateReportColumn`), NOT at compile/execute time. This means legacy formulas that predate the blocklist continue to work.

`sanitizeFormula()` in `utils/report-engine.js` blocks:
- **Keywords:** SELECT, UNION, DROP, DELETE, INSERT, UPDATE, ALTER, CREATE, TRUNCATE, GRANT, REVOKE, COPY, EXECUTE, EXEC, pg_read_file, pg_read_binary_file, pg_ls_dir, lo_import, lo_export, current_setting, dblink, dblink_exec
- **SQL comments:** `/*`, `*/`, `--`
- **Dollar-quoting:** `$$`
- **Semicolons**

NOT blocked: `FROM` (used in `EXTRACT(MONTH FROM [THIS])`).

**Legacy:** workspace a2025 has a formula containing `SELECT DISTINCT...`. It works because validation was moved from compile-time to write-time. Re-saving that formula will be blocked.

## Column Config

```json
{
  "reqTypeId": 42,
  "alias": "Total",
  "displayName": "Итого руб.",
  "func": "SUM",
  "totalFunc": "SUM",
  "storedFrom": "2024-01-01",
  "storedTo": "[TODAY]",
  "havingFrom": 1000,
  "havingTo": null,
  "setExpr": "=[Цена] * [Кол-во]"
}
```

- `func`: `SUM` | `AVG` | `COUNT` | `MIN` | `MAX` | `GROUP_CONCAT` | `STRING_AGG`. System functions: `ABN_ID`, `ABN_UP`, `ABN_TYP`, `ABN_ORD`, `ABN_REQ`, `ABN_BT`, `ABN_URL`, `ABN_ROWNUM`, `ABN_DOMAIN`, `ABN_TAG`. Transform functions: `DATE_TRUNC`, `UPPER`, `LOWER`, `LENGTH`, `TRIM`, `ROUND`, `EXTRACT`, `COALESCE`, `ABS`, `CEIL`, `FLOOR`.
- `totalFunc`: footer aggregation shown at bottom of report. Valid values: `SUM`, `AVG`, `COUNT`, `MIN`, `MAX`, `GROUP_CONCAT`, `STRING_AGG`, `NONE`. Set to `null` or `NONE` to suppress the footer total for that column.
- `displayName`: human-readable column header (overrides auto-generated name). Used for **column renaming** via `PATCH /reports/:reportId/columns/:colId` with `{ displayName: "new name" }`. Dot-notation (`"руб/ед.Комус"`) also triggers smart two-level header grouping in DataTable.
- `setExpr`: SET expression for bulk updates via `report_bulk_update` (e.g. `=[Цена] * [Кол-во]`). Column references resolved at runtime.
- `sortDir`: column sort direction — `ASC` or `DESC`
- `sortPriority`: sort priority (1-9) — lower number = higher priority when multiple columns are sorted
- `format`: column display format — `AUTO` | `HTML` | `MEMO` | `FILE` | `DATE` | `DATETIME` | `NUMBER` | `BOOLEAN`
- `joinAlias`: alias for a joined table column (references a JOIN)
- `hidden`: if `true`, column is excluded from visible output but still participates in filtering/grouping
- `order`: column display order (integer)
- `storedFrom/To`: default date range filter (pre-applied when report opens)
- Tokens in date fields: `[TODAY]`, `[NOW]`, `[USER]`, `[USER_ID]`

## Run Parameters

```json
{
  "filters": { "Status": { "from": "Active", "to": null, "eq": "Active", "like": "%draft%" } },
  "where": "AND c123.val = '[USER]'",
  "order": "c456 DESC",
  "limit": 100,
  "offset": 0,
  "totals": "c123:SUM,c456:AVG",
  "format": "default|kv|columnar|cr|hr|count",
  "select": "1,2,3",
  "fieldNames": "true",
  "filterId": 42
}
```

- `select`: comma-separated string of column IDs to include in the result (subset of report columns)
- `fieldNames`: string flag; if `"true"`, include field names in the response
- `filterId`: apply a saved filter by ID
- Filter operators: `from`/`to` (range), `eq` (exact match), `like` (pattern match with `%` wildcards)

### URL Filter Params

Report run also accepts filters via query string parameters, merged with body filters:

- `FR_<colId>` — `from` filter value for column
- `TO_<colId>` — `to` filter value for column
- `F_I` — filter by object IDs (comma-separated)
- `F_U` — filter by `up` (parent ID)

## CSV Export

Streaming response. Headers sent immediately; rows streamed in chunks. On mid-stream error, socket is destroyed to signal truncation to the client.

## SSE/NDJSON Stream

`GET /reports/:reportId/stream?format=sse|ndjson&batchSize=500`. Sends rows in batches. Useful for reports with millions of rows. SSE sends `event: done` at the end.

## Report Engine (`utils/report-engine.js`)

The SQL engine that translates report column config into parameterized SQL: multi-table JOINs via EAV `_v2_reqs`, GROUP BY, HAVING, WHERE interpolation, column aliases. ~1,559 lines.

### Aggregate reports (GROUP BY)

When a report has aggregate columns (SUM, COUNT, AVG, etc.):

- `a.id`, `a.val`, `a.up`, `a.ord` are wrapped in `MIN()` in SELECT — they must NOT appear as bare expressions in GROUP BY, because `a.id` is unique per row and would prevent any actual aggregation.
- Only non-aggregate, non-main, non-hidden columns go into GROUP BY.
- **Hidden columns** in aggregate reports are wrapped in `MIN()` in SELECT and excluded from GROUP BY entirely. This prevents hidden columns (e.g. a Status ref used only for filtering) from splitting the grouping. Their `__raw` expression is also skipped.
- For ref columns in GROUP BY, all sub-expressions must be listed: `"alias".val`, `"alias_rn".val`, and the `__raw` CASE WHEN expression.
- ORDER BY defaults to `1` (first column) instead of `a.ord` for aggregate reports.
- The `countOnly` path uses the full SELECT + GROUP BY to count distinct groups.
- **Formula columns in GROUP BY** use `_resolvedExpr` (the formula-wrapped expression), not `_rawExpr`. This allows `DATE_TRUNC('month', [THIS]::timestamp)::date` formulas to group by the truncated value rather than the raw datetime.

### V2 extended types and `isRef` detection

`colIsRef` determines whether a column is a foreign-key reference (needs `_rn` JOIN for display name resolution). V2 extended types like CURRENCY (1002), PERCENT (1003), etc. are value types — NOT references. They are excluded via `REV_BASE_TYPE[col_base_t]` check, even though their EAV type chain has `col_base_t !== col_base_base_t`.

## Report Structure Cache

`getReport()` caches the parsed report structure in `reportCache` (in-memory LRU). The cache is invalidated on every write operation: `addReportColumn`, `reorderReportColumns`, `updateReportColumn`, `deleteReportColumn`, `addReportJoin`, `deleteReportJoin`, and `updateReport`. Key format: `report:<db>:<reportId>`.

## Idempotency

`X-Idempotency-Key` supported on POST create, cached 30 seconds.

## DB Tables

- `_v2_report_columns` (per-workspace) — column definitions per report
- `_v2_report_joins` (per-workspace) — JOIN definitions
- `_v2_reports` stored in global schema or per-workspace depending on init
- `_v2_report_audit` (per-workspace, lazy-init) — change log for report structure

## Inline Editing in ReportView

Reports backed by a single non-aggregate type support inline row creation and cell editing directly in the report UI (no navigation to object page).

**Behaviour:**
- `clickNavigates = false` always — clicking a cell opens inline editor, never navigates
- `+ Создать запись` button visible when `canWrite && !hasAggregates` (even on empty report)
- New row created via `objectsService.create()`, added inline to `reportRows` without page reload
- Cell save: main column (isMainCol) updates via `body.value`; other columns via `body.requisites`

**Column name parsing** (`service.js → cleanName()`): prioritizes `:ALIAS=name:` marker (highest priority — returns alias directly), then strips `:ICON=...:` and `:VALCOL=...:` markers from type row val to produce human-readable column header.

## Script Button Columns in Reports

Reports can include Script Button columns (type 1020). When a user clicks Run in a report row:
1. `onScriptButtonRun` in `ReportView.vue` calls `scriptButtonService.run()`
2. On success, `runReport()` refreshes the report to show updated values
3. Toast shows the written column name and value

The script sandbox supports `setField(reqId, value)` to write to **multiple** columns in one click — useful for filling Ед., Мин. цена, Мин. сумма simultaneously from one script run.

### FORMAT_TO_TYPE mapping

`frontend/src/utils/report-formats.js` maps backend format strings to numeric column types:
- `'SCRIPT_BUTTON'` → `1020`
- `'AI_BUTTON'` → `1015`
- `'HTTP_BUTTON'` → `1016`
- `'AI_COLUMN'` → `1018`

## Matrix View

Route: `/reports/:reportId/matrix` (frontend `MatrixReportView.vue`).

A pivot-table view of a report. Uses `pivot.js` utility to transform flat report rows into a matrix with row/column headers.

**Features:**
- Plan/Fact/Delta cells — optional fact report overlay via `mergePlanFact`, showing plan value, fact value, and computed delta
- Formula rows — virtual rows computed from matrix data (`computeFormulaRows`), configured via URL query param `?formulas=[...]`
- Inline cell editing — click a cell to edit its value directly in the matrix

## `createReportSchema` Missing Fields

`createReportSchema` (Zod) accepts only `name`, `parentType`, `storedLimit`, `repUrl`. It does **not** include `icon` or `where` (both present in `updateReportSchema`). The `create_report` AI tool TOOL_DEF does accept `icon` and `where` params — these are silently stripped when the REST path validates via Zod. To set `icon` or `where` on a new report, a follow-up `PATCH` (or `update_report` tool call) is required.

## AI Tools

| Tool | Tier | Description |
|------|------|-------------|
| list_reports | TIER_LOW | List reports in workspace |
| get_report | TIER_LOW | Get report by ID |
| describe_report | TIER_LOW | Describe report structure |
| create_report | TIER_MEDIUM | Create a new report |
| update_report | TIER_MEDIUM | Update report config |
| delete_report | TIER_HIGH | Delete a report |
| add_report_column | TIER_MEDIUM | Add column to report |
| update_report_column | TIER_MEDIUM | Update report column |
| delete_report_column | TIER_HIGH | Delete report column |
| reorder_report_columns | TIER_MEDIUM | Reorder report columns |
| get_report_history | TIER_LOW | Get report audit log |
| create_report_join | TIER_MEDIUM | Add JOIN to report |
| delete_report_join | TIER_HIGH | Delete report JOIN |
| export_report | TIER_LOW | Export report data |
| report_bulk_update | TIER_HIGH | Bulk update records via report |
