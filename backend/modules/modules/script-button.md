# Module: script-button

`src/api/v2/modules/script-button/`

## Purpose

Script Button is a column type (TYPE=1020) that lets users write JavaScript code that runs on button click — similar to Airtable's "Run script" button. The script receives the current row's field values via `row`, can make HTTP requests with `fetch`, and writes a result back to any column via `output()`.

## Execution Model

Scripts execute **exclusively in the browser** in a Web Worker. Server-side execution has been removed for security (Node.js `vm` is not a security boundary).

When a user clicks a script button in the UI, the script executes in a **Web Worker** in the browser. This provides:
- Thread isolation (separate thread, no DOM access)
- Timeout enforcement (60s, `worker.terminate()`)
- No server-side code execution risk (no vm escape, no SSRF from user scripts)

HTTP requests from scripts go through a **server-side fetch proxy** (`POST /fetch-proxy`) to bypass CORS. The proxy blocks requests to internal/private networks (SSRF protection).

**Flow:** CellRenderer click → ObjectList.vue `onScriptButtonRun` → loads config → builds `row` → `runScriptInWorker()` → Web Worker executes script → result written to outputReqId via `objectsService.update`.

**Files:**
- `frontend/src/utils/scriptRunner.js` — orchestrator, creates Worker, relays postMessage
- `frontend/src/workers/script-worker.js` — Web Worker, sandboxed globals, `new Function()` execution

**No server-side execution:** `run_script_button` has been removed from automations and MCP tools. Scripts can only run via the browser UI button.

## Files

| File | Description |
|------|-------------|
| `router.js` | Express routes — GET config, POST config, POST fetch-proxy. POST /run returns 410 Gone. |
| `service.js` | Config CRUD only (`getConfig`, `saveConfig`, `initTable`). No script execution. |

## Config Table

`_v2_column_script_config` — one row per column (typeId + reqId pair):

```sql
CREATE TABLE IF NOT EXISTS "<db>"._v2_column_script_config (
  id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  type_id       INT NOT NULL,
  req_id        INT NOT NULL,
  script        TEXT NOT NULL DEFAULT '',
  output_req_id INT,
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE (type_id, req_id)
)
```

## Script Globals

Available in browser Web Worker execution:

| Global | Description |
|--------|-------------|
| `row` | Object mapping field names to values: `row['Название']`, `row['Цена']`, etc. Also includes `row.ID` (object id as string) and `row.VAL` (display name) |
| `fetch` | HTTP requests proxied through server-side fetch-proxy (CORS bypass, SSRF-protected) |
| `ai(prompt, model?)` | Async LLM call. `model` defaults to `'fast'`. Returns the response text string. |
| `output(value)` | Set result value; written to `outputReqId` column if configured |
| `setField(reqId, value)` | Write `value` to any column by reqId |
| `createDocument(title, sections)` | Create a formatted document. `sections` is an array of `{ title, content }` (content is markdown). Returns `{ id, url }`. |
| `JSON` | Standard JSON |
| `Math` | Standard Math |
| `Date` | Standard Date |
| `console` | `log`, `warn`, `error` — captured in logs array, forwarded to main thread |

### Example scripts

```js
// Simple string manipulation
output(row['Название'] + ' — processed');

// HTTP fetch
const res = await fetch('https://api.example.com?sku=' + row['Артикул']);
const data = await res.json();
output(data.price);

// AI call (fast model)
const summary = await ai('Summarize in one sentence: ' + row['Описание']);
output(summary);

// AI call with smart model
const analysis = await ai('Classify this product: ' + row['Наименование'], 'smart');
output(analysis);

// Create a document from fields
const doc = await createDocument('Report: ' + row['Name'], [
  { title: 'Summary', content: row['Summary'] },
  { title: 'Analysis', content: row['Analysis'] },
]);
output('Document created: ' + doc.url);
```

## REST Endpoints

All routes under `/:db/script-button`, protected by `requireJwt` + workspace role middleware.

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/config` | viewer | Get config for a column. Query: `typeId`, `reqId` |
| POST | `/config` | editor | Save/update config. Body: `{ typeId, reqId, script, outputReqId }` |
| POST | `/run` | editor | **Disabled.** Returns 410 Gone. Server-side execution removed. |
| POST | `/fetch-proxy` | editor | Server-side fetch proxy for browser scripts. SSRF-protected. Body: `{ url, method, headers, body }` |

### POST /fetch-proxy

Proxies HTTP requests from browser-executed scripts to bypass CORS restrictions.

**SSRF protection:** Blocks requests to `localhost`, `127.0.0.1`, `::1`, `0.0.0.0`, `*.local`, `*.internal`, and private IP ranges (10.x, 172.16-31.x, 192.168.x, 169.254.x).

**Response:**
```json
{ "status": 200, "statusText": "OK", "headers": {...}, "body": "..." }
```

## Browser Sandbox Security

The Web Worker uses a two-layer approach to block dangerous globals from user scripts:

1. **Global scope override** — `eval` and `Function` are set to `undefined` on the Worker's `self` before script execution (they cannot be used as `new Function()` parameter names in strict mode). Originals are saved and restored after execution.

2. **Function parameter shadowing** — all other dangerous globals are passed as `undefined` parameters to the `new Function()` wrapper. This shadows the outer scope without modifying getter-only properties (like `indexedDB`).

**Blocked globals:** `self`, `globalThis`, `eval`, `Function`, `importScripts`, `XMLHttpRequest`, `WebSocket`, `indexedDB`, `navigator`, `location`, `setTimeout`, `setInterval`, `clearTimeout`, `clearInterval`, `requestAnimationFrame`, `cancelAnimationFrame`, `Notification`, `BroadcastChannel`.

**Allowed globals:** `row`, `fetch` (proxied), `ai`, `output`, `setField`, `console` (captured), `JSON`, `Math`, `Date`.

All user code runs in strict mode (`"use strict"`) inside an async IIFE.

## Frontend Integration

- Column type registered as `SCRIPT_BUTTON = 1020` in `backend/src/shared/constants.js` and `frontend/src/constants/columnTypes.js`
- `CellRenderer.vue` renders a purple "Run" button (icon: `pi-code`) with spinner for type 1020
- `ScriptButtonConfigDialog.vue` — config form: JS editor (monospace textarea), available fields list, output column selector
- `ObjectList.vue` — manages `scriptPendingCells` Set, `onScriptButtonRun` (browser-side via Web Worker), `onScriptButtonConfig`
- `ReportView.vue` — script button support in report views (кнопки отображаются и выполняются в контексте отчёта)
- `DataTable.vue` — passes `scriptPendingCells` prop to CellRenderer, emits `script-button-run` and `script-button-config`
- `frontend/src/services/scriptButton.js` — axios wrapper for config and fetchProxy endpoints
- `frontend/src/utils/scriptRunner.js` — Web Worker orchestrator
- `frontend/src/workers/script-worker.js` — sandboxed script execution in Web Worker

## Workspace Init

`initTable(pool, db)` is called from `ensureAllSatelliteTables` in `workspaces/service.js` on workspace creation and clone. Also called lazily at the start of `getConfig` and `saveConfig` (idempotent — `CREATE TABLE IF NOT EXISTS`).
