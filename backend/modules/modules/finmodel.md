# Module: finmodel

**Path:** `src/api/v2/modules/finmodel/`
**Files:** `router.js`, `service.js`, `engine.js`, `periods.js`, `types.js`, `schema.js`
**Base URL:** `/api/v2/:db/finmodel/...` (gated by workspace module `finmodel`, default ON)
**Auth:** JWT; reads — any member, writes — editor + CSRF, `setup` — admin + CSRF. MCP: `fin_grid`/`fin_graph` TIER_LOW, `fin_goal_seek` TIER_MEDIUM (it writes cells when `apply: true`).

## Purpose

Financial models with **calculation groups**: a sheet of rows grouped into blocks, values spread over periods (year/quarter/month/week), several scenarios per model, formulas that reference other rows with period offsets, and **goal seek** — pick input values so that a target cell reaches a wanted number.

Everything is stored as ordinary EAV records, so a model is visible, filterable and shareable like any other data: Финмодель → Лист → Панель → Расчётная группа → Строка → Ячейка, a chain of child tables linked by `up`. No ref columns are used at all — ownership is `up`, which avoids the direct/inverted ref split (`.claude/rules/backend.md`).

## Endpoints

| Method | Path | Role | Description |
|--------|------|------|-------------|
| POST | `/finmodel/setup` | admin + CSRF | Create the six tables and their columns; idempotent. Returns `{created, kept, types}` |
| GET | `/finmodel` | any member | List models |
| POST | `/finmodel` | editor + CSRF | Create a model `{name, from, to, granularity, scenarios?, description?}` |
| GET | `/finmodel/:id` | any member | Model row + structure |
| PATCH | `/finmodel/:id` | editor + CSRF | Change period range, granularity, scenarios, description |
| DELETE | `/finmodel/:id` | editor + CSRF | Delete the model with everything under it |
| GET | `/finmodel/:id/grid?scenario=&sheetId=` | any member | Computed grid: `columns`, `rows` (cells, totals, `source` per cell), `structure` |
| GET | `/finmodel/:id/graph?scenario=&sheetId=` | any member | Cell dependency graph: `order`, `edges`, `cycle` |
| POST | `/finmodel/:id/nodes` | editor + CSRF | Add a sheet / panel / group / row (`{level, parentId, name, kind?, formula?, unit?, config?}`) |
| PATCH | `/finmodel/:id/nodes/:nodeId` | editor + CSRF | Rename, change kind / formula / unit / group config, or move the node with `ord` |
| DELETE | `/finmodel/:id/nodes/:nodeId` | editor + CSRF | Delete a node with its subtree |
| POST | `/finmodel/:id/cells` | editor + CSRF | Write up to 2000 cells `{rowId, period, value}`; `value: null` erases |
| POST | `/finmodel/:id/goal-seek` | editor + CSRF | Pick lever values for a target cell; writes only with `apply: true` |

`granularity`: `year`, `quarter`, `month`, `week`. Column keys are `2027`, `2027-Q1`, `2027-03`, `2027-W09`.

Before setup every route except `setup` answers **409 `FINMODEL_NOT_SET_UP`** and names what is missing — an empty model list would tell the user something untrue.

## MCP tools

- `fin_grid` — computed grid for a scenario (group subtotal rows come back with ids like `g123`)
- `fin_graph` — dependency graph; a cycle is **returned**, not refused, because seeing it is the point
- `fin_goal_seek` — goal seek, with `apply` to write the solution

## How It Works

1. **Structure lives in EAV.** Tables are found **by name** (`resolveFinTypes`), not through `_v2_template_origin`: the map is filled only by applying a workspace template, and a workspace set up via `POST /setup` is not in it at all. Price of the decision: renaming a table from the UI breaks the link, and the next `setup` creates a new table beside it — that is why `setup` reports `created` and `kept` instead of staying silent.
2. **Group kinds** (`GROUP_KIND` in `types.js`): `value` — rows entered by hand, `formula` — rows computed, `sum_rows` — hand-entered rows plus a synthetic «Итого <Группа>» row. A row's own kind (`ROW_KIND`) overrides the group's; an empty row kind means "as the group".
3. **Formulas** reference rows by name, by `Панель.Строка`, or by id, with an optional period offset: `[Накопленным итогом][-1] + [Поток]`. Aggregates: `СУММА(...)` (including ranges `СУММА([А]:[Б])`), `ИТОГО`, `НАКОПИТЕЛЬНО`.
4. **The dependency graph is over cells, not rows** (`engine.js`): a row referencing itself one period back is legal and must not be declared a cycle. A real cycle raises `CycleError` → 422 with the participants listed; the platform's `topoSort` skips cycles silently, which is unacceptable here — a skipped cell looks like a plausible number.
5. **Unknown ≠ zero.** `undefined` — no such (row, period); `null` — the cell exists but its value is unknown; a number — a measured quantity, zero included. One unknown addend makes the whole formula unknown; a sum is unknown only if **all** addends are. The single exception is an offset that leaves the model range — that is "no such period", counted as nothing and marked `outOfRange` so the grid can show it.
6. **Scenarios.** A cell is either scenario-specific or shared by all scenarios. Writing over a shared cell is refused (409 `FINMODEL_SHARED_CELL`) instead of quietly forking it; the grid marks each cell with `scenarioSpecific` so the UI knows what it edits.
7. **Goal seek** (`goalSeek`): secant method along **one** direction. The task is underdetermined — many levers, one condition — so the direction is weighted by measured sensitivity (the shortest correction that produces the needed change in the linear approximation), which keeps the search from dumping everything onto the first lever. Defaults: 40 iterations (max 200), tolerance relative to the target (`max(1e-6, |target|·1e-9)`). Levers must be input rows; the target cell cannot be its own lever. Nothing is written unless `apply: true` **and** the target was reached — `reachable`/`applied` say so explicitly. Unreachable outcomes are named: `unknown_target`, `no_sensitivity`, `flat`, `bounds`, `no_convergence`.
8. **Reordering is a separate action** inside the same PATCH: `updateObject` knows nothing about order, so `ord` is handed to `reorderObject`, which shifts same-parent siblings in one transaction. That is what makes drag-and-drop on the sheet honest — the number means the place from top to bottom, not a sparse weight.
9. **Writes go through the objects service** (`createObject`/`updateObject`/`deleteObject`), so grants, row rules and autofields apply exactly as they do to any other record — including goal seek, which writes through `writeCells` rather than a second writer of its own.

## DB Tables

None of its own: six EAV tables inside the workspace data table — «Финмодель», «Лист финмодели», «Панель финмодели», «Расчётная группа», «Строка финмодели», «Ячейка финмодели». Declared once in `types.js`; the `vc-fund` workspace template carries the same shape and `manifest-matches-types.test.js` keeps the two in step.

## Known MVP Limits

- Group kinds are three (`value`, `formula`, `sum_rows`); report/plan-fact style groups are not built.
- Vocabularies (kinds, granularities) live in code, not in EAV lookups — a closed list the arithmetic depends on must not be editable as data.
- Tables are matched by name, so renaming one from the UI disconnects the module until `setup` runs again.
- The frontend covers sheets, panels, groups, rows, inline cell editing, the formula dialog, drag reordering and goal seek (`frontend/src/views/finmodel/`, `frontend/src/components/finmodel/`). Not on the screen yet: editing the model itself after creation (period range, granularity, scenarios — `PATCH /finmodel/:id` only), moving a row into another group (that changes its kind, so it is a different action), and export — the grid copies a selection as TSV, nothing more.
