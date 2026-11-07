# Module: http-button

`src/api/v2/modules/http-button/`

## Purpose

HTTP Button is a column type (TYPE=1016) that lets users trigger direct HTTP requests from a table cell — no LLM, deterministic, fast. Analogous to Airtable's button→automation→script pattern but without a scripting layer: config defines URL, method, headers, body, and where to write the response.

## Files

| File | Description |
|------|-------------|
| `router.js` | Express routes — GET config, POST config, POST run |
| `service.js` | Business logic — config CRUD, placeholder interpolation, fetch, response extraction |

## Config Table

`_v2_column_http_config` — one row per column (typeId + reqId pair):

```sql
CREATE TABLE IF NOT EXISTS "<db>"._v2_column_http_config (
  id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  type_id     INTEGER NOT NULL,
  req_id      INTEGER NOT NULL,
  config      TEXT    NOT NULL DEFAULT '{}',
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (type_id, req_id)
)
```

Config JSON shape:
```json
{
  "method": "GET",
  "url": "https://api.example.com/price?sku=[Артикул]",
  "headers": { "Authorization": "Bearer token" },
  "bodyTemplate": "",
  "responsePath": "data.price",
  "outputReqId": 42
}
```

## Placeholder Interpolation

`[ColumnName]` tokens in `url`, `headers`, and `body` are replaced with the current row's value for the column with that name. The column lookup is case-sensitive and matches the requisite `name` field.

Example: `url = "https://example.com?q=[Название]"` with row value `"Дрель"` → `"https://example.com?q=Дрель"`.

## Response Path

`responsePath` is a dot-notation path into the JSON response body. Empty or absent means use the entire response body as a string.

Examples:
- `""` → raw response body
- `"price"` → `response.price`
- `"data.items.0.value"` → nested access

## Output Column

`outputReqId` is the column ID to write the extracted value into. If null/absent, the value is returned in the API response but not persisted.

## REST Endpoints

All routes under `/:db/http-button`, protected by `requireJwt` + workspace role middleware.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/config` | Get config for a column. Query: `typeId`, `reqId` |
| POST | `/config` | Save/update config. Body: `{ typeId, reqId, method, url, headers, bodyTemplate, responsePath, outputReqId }` |
| POST | `/run` | Run the button for a row. Body: `{ typeId, reqId, objectId }` |

### POST /run response

```json
{ "value": "12500", "writtenTo": "Цена", "outputReqId": 42 }
```

- `value` — extracted string value
- `writtenTo` — output column name (null if no outputReqId configured)

## Frontend Integration

- Column type registered as `HTTP_BUTTON = 1016` in `src/shared/constants.js` and `frontend/src/constants/columnTypes.js`
- `CellRenderer.vue` renders a green "Run" button with spinner for type 1016
- `HttpButtonConfigDialog.vue` — full config form (method, URL, headers, body, response path, output column)
- `ObjectList.vue` — manages `httpPendingCells` Set, `onHttpButtonRun`, `onHttpButtonConfig`
- `DataTable.vue` — passes `httpPendingCells` prop to CellRenderer, emits `http-button-run` and `http-button-config`
- `frontend/src/services/httpButton.js` — axios wrapper for all three endpoints

## Workspace Init

`initTable(pool, db)` is called from `ensureAllSatelliteTables` in `workspaces/service.js` on workspace creation and clone.
