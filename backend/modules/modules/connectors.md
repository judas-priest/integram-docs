# Module: connectors

**Path:** `src/api/v2/modules/connectors/`
**Files:** `router.js`, `service.js`, `presets.js`, `cdek/`, `pochta/`, `smsru/`
**Base URL:** `/api/v2/:db/connectors/...`
**Auth:** JWT required. Create/update/delete: `admin`. Run: `editor`. List/get: any.

## Purpose

Integrations with external HTTP APIs. Each connector has a JSONB config describing the API endpoint, authentication, field mappings, and import rules. Execution fetches data from the external API and creates/updates EAV objects. Connectors run synchronously (up to 10 minutes per request).

## Endpoints

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/connectors` | any | List connectors |
| POST | `/connectors` | admin | Create connector |
| GET | `/connectors/presets` | any | List built-in preset configs |
| GET | `/connectors/:id` | any | Get connector |
| PATCH | `/connectors/:id` | admin | Update connector config |
| DELETE | `/connectors/:id` | admin | Delete connector |
| POST | `/connectors/:id/run` | editor | Execute connector (fire-and-forget, tracks progress via notification) |
| POST | `/connectors/:id/discover` | admin | Discover OData schema from connector's configured URL |
| POST | `/connectors/cancel` | editor | Cancel a running connector by `notifId` |

## Connector Config Structure

```json
{
  "url": "https://api.example.com/data",
  "method": "GET",
  "auth": { "type": "bearer", "token": "{{TOKEN}}" },
  "headers": {},
  "params": { "company": "{{COMPANY_ID}}" },
  "pagination": { "type": "page", "pageParam": "page", "limitParam": "limit", "location": "body" },
  "mapping": { "response.path": reqId, ... },
  "targetTypeId": 123,
  "incremental": { "cursorField": "updatedAt", "cursorParam": "updatedAfter" },
  "incrementalMode": "stop-when-seen",
  "writeMode": "upsert"
}
```

### Auth formats

- `{ "type": "bearer", "token": "..." }` — `Authorization: Bearer <token>`
- `{ "type": "basic", "username": "...", "password": "..." }` — `Authorization: Basic <base64>`
- `{ "type": "basic", "credentials": "login:password" }` — single interpolatable credentials string
- `{ "type": "apikey", "header": "X-Api-Key", "value": "..." }` — custom header
- `{ "type": "google_service_account", "credentials": "{{service_account_json}}" }` — Google OAuth2 JWT flow. Signs a JWT (RS256) with the private key from the Service Account JSON, exchanges it for an access_token at `https://oauth2.googleapis.com/token`. Token cached in memory with 1-min early-refresh buffer (~59 min effective lifetime). Scopes: `documents`, `spreadsheets`, `drive`.

### Interpolation

`{{VAR}}` placeholders in `url`, `headers`, and `params` are substituted from `config.params` at runtime. This lets templates define connector shapes without hardcoding secrets — tokens are filled via `POST /workspaces/:slug/fill-connector-tokens`.

### Incremental sync

`incremental: { cursorField, cursorParam }` — enables cursor-based incremental sync:
- `cursorField` — JSON path in each fetched item to read the cursor value (e.g. `"updatedAt"`)
- `cursorParam` — query param name to pass the cursor to the API (e.g. `"updatedAfter"`)
- Cursor state persisted in `_v2_connector_state` per connector
- Pass `fullResync: true` in run body to clear the cursor and fetch all data

`incrementalMode: "stop-when-seen"` — stops pagination early when an entire page contains only already-known records (no new inserts). Useful for APIs that return newest-first.

### Write mode

`writeMode` controls how existing records are handled during import:

- `"upsert"` (default) — update existing records matched by `idField`, create new ones if not found
- `"append"` — always create new records, never update existing ones

### Body-based pagination

Set `pagination.location: "body"` for APIs that accept pagination params in the JSON request body (e.g. Ozon v3). When set, pagination params (`cursorParam`, `pageParam`, `limitParam`) are merged into the body JSON on each page request instead of being appended to the query string.

### Detail fetching (per-item)

When a list API returns only summary data, use detail fetching to call a secondary URL per item:

| Field | Type | Description |
|-------|------|-------------|
| `detailUrl` | string | URL template for per-item detail fetch (e.g. `https://api.example.com/items/{{id}}`) |
| `detailIdField` | string | Field in the list item to substitute into `detailUrl` (default: `idField`) |
| `detailDataPath` | string | Dot-path to extract from the detail response |
| `detailDelay` | int (ms) | Delay between detail requests to avoid rate limiting |

Detail data is merged with the list item before mapping is applied.

### Field-level `skipIfExists`

When a field mapping object has `"skipIfExists": true`, the connector will **not** overwrite existing field values on update. New records still get the field written. Use when multiple data sources feed the same table and one source is more authoritative (e.g. UDS fills sum/address accurately; Sheets should not overwrite them).

```json
{
  "sum": { "reqId": 340, "skipIfExists": true },
  "address": { "reqId": 339, "skipIfExists": true }
}
```

### CSRF flow (SAP)

Set `csrfFlow: true` for SAP APIs that require x-csrf-token preflight. On POST/PUT/PATCH requests, the connector first sends a HEAD/GET to fetch the CSRF token, then includes it in the mutation request.

## Execution Flow

1. Router creates "running" notification, registers `AbortController`, returns immediately with `{ status: 'running', notifId }`
2. `service.executeConnector()` runs in the background: fetches data from external API, applies field mapping, creates/updates EAV objects
3. For each **newly created** object: `bus.emit('object.created', ...)` fires via `sideEffect` — triggers `on_create` automations (e.g. Telegram notifications)
4. For each **updated** object where at least one field value actually changed: `bus.emit('object.updated', { changedReqIds })` fires — triggers `on_update` automations (e.g. status change notifications). Old vs new value is compared per field; no event if nothing changed.
5. On completion or error, router updates the notification to "done", "error", or "cancelled" (on abort)
5. `AbortController` enables mid-run cancellation via `POST /connectors/cancel`

## Presets

Static JSON configs for common integrations. Returned by `GET /connectors/presets`.

Available presets: `1c_odata`, `1c_http`, `1c_product`, `sap_rest`, `uds`, `uds_partner`, `moysklad`, `scada_rest`, `bitrix24`, `bitrix24_product`, `wildberries`, `amocrm`, `hhru`, `tbank`, `ozon`, `google_sheets_read`, `google_sheets_append`, `google_docs_create`, `cdek`, `pochta`, `vk_market`, `sms_ru`, `alfabank`.

**ADR-016 Object Layer presets:**
- `1c_product` — 1C Nomenklyatura import to canonical Product
- `bitrix24_product` — Bitrix24 catalog products to canonical Product

**Google Workspace presets** (`google_sheets_read`, `google_sheets_append`, `google_docs_create`) use `auth.type = "google_service_account"`. Require a Service Account JSON from Google Cloud Console with the target spreadsheet/document shared with the service account email.

## CDEK Submodule (`cdek/`)

Special handling for CDEK (Russian courier) API v2: delivery cost calculation, shipment creation, and webhook-based status tracking.

**Files:** `token.js` (OAuth2), `client.js` (API wrapper), `actions.js` (business logic), `router.js` (CRM routes)

### Authentication

OAuth2 client credentials flow. Base URLs:
- Production: `https://api.cdek.ru/v2`
- Test: `https://api.edu.cdek.ru/v2`

Token cached per `clientId:testMode`, refreshed 60s before expiry.

### CRM Routes

| Method | Path | Role | Description |
|--------|------|------|-------------|
| POST | `/:db/cdek/shipment` | editor | Create CDEK shipment from order |
| GET | `/:db/cdek/points?city=<code>` | editor | List pickup points by city code |
| GET | `/:db/cdek/label/:orderId` | editor | Download barcode label PDF for shipment |

### Portal Routes (in `portal/router.js`)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/:db/portal/api/delivery/calculate` | none (rate-limited) | Calculate tariffs for checkout |
| POST | `/:db/portal/api/cdek/webhook/:secret` | secret validation | Receive CDEK status webhook |

### Webhook auto-registration

When a CDEK connector is created or updated (`type: 'cdek'`), `autoRegisterCdekWebhook()` is called as a side-effect. It registers the webhook URL `{PUBLIC_URL}/api/v2/{db}/portal/api/cdek/webhook/{webhookSecret}` with CDEK API. Requires `PUBLIC_URL` env var.

### Webhook status mapping (`handleWebhook`)

When `ORDER_STATUS` webhook arrives, maps CDEK status codes to Integram order statuses if `orderStatusReqId` is set in connector config:

| CDEK code | Integram status |
|---|---|
| ACCEPTED, RECEIVED_AT_SENDER_WAREHOUSE, READY_FOR_SHIPMENT_IN_SENDER_CITY, TAKEN_BY_TRANSPORTER_FROM_SENDER | 🚚 В доставке |
| RETURNED_TO_SENDER_CITY | 🔴 Отменён |
| READY_FOR_PICKUP | 📍 В пункте выдачи |
| DELIVERED | ✅ Доставлен |

**Connector config fields:**
- `clientId`, `clientSecret` — CDEK API credentials
- `testMode` — use edu.cdek.ru instead of api.cdek.ru
- `webhookSecret` — secret for webhook URL validation
- `senderCity`, `senderAddress` — origin for shipments
- `defaultTariffCode` — tariff for shipment creation
- `ordersTypeId` — EAV type ID of orders table
- `trackingReqId`, `cdekUuidReqId`, `cdekNumberReqId` — tracking fields
- `orderStatusReqId` — ref column for status; resolved to lookup record ID
- `pvzFactReqId` — PVZ address field; written on `READY_FOR_PICKUP`
- `weightReqId`, `lengthReqId`, `widthReqId`, `heightReqId` — dimension fields for shipment creation
- `recipientNameReqId`, `recipientPhoneReqId`, `recipientAddressReqId`, `recipientCityReqId` — recipient fields
- `sumReqId` — Deprecated — no longer used. Item cost is hardcoded to 1000.
- `linkedOrderReqId` — Ref requisite ID for linked orders — used to propagate CDEK webhook status changes to linked orders
- `_deliveryPoint` — internal flag for PVZ-mode shipments

### Status reconciliation (`reconcileStatuses`)

Polls CDEK API for orders with `cdekUuidReqId` that are currently in "В доставке" status. Updates status if CDEK reports terminal/significant statuses. Uses `RECONCILE_STATUS_MAP` (subset: `DELIVERED`, `READY_FOR_PICKUP`, `RETURNED_TO_SENDER_CITY`). Rate-limited at 300ms per order.

Called by automation action `reconcile_cdek` and AI tool `reconcile_cdek`.

## Pochta Rossii Submodule (`pochta/`)

Russian Post tracking via SOAP API. Polling-based — no webhooks.

**Files:** `client.js` (SOAP client), `actions.js` (polling logic), `router.js` (CRM routes)

**API:** `https://tracking.russianpost.ru/rtm34` (SOAP 1.2, `getOperationHistory`)

### How it works

1. `pollTracking()` finds active orders with tracking numbers and delivery method = Почта России
2. For each order, calls SOAP API to get operation history
3. Maps the latest operation type to Integram order status
4. Updates EAV fields (status, PVZ address) and fires `on_update` automation

### CRM Routes

| Method | Path | Role | Description |
|--------|------|------|-------------|
| POST | `/:db/pochta/poll` | editor | Poll tracking for all active orders |
| GET | `/:db/pochta/track?barcode=<num>` | editor | Get tracking history for single barcode |

### Execution via `run_connector`

When `executeConnector()` encounters a config with `type: 'pochta'`, it delegates to `pollTracking()` instead of the HTTP flow. This means Pochta connectors can be triggered by automations via `run_connector` action (e.g. scheduled every 30 minutes).

### Status mapping

| Pochta operation type | Integram status |
|---|---|
| 1 (Приём), 2 (Обработка), 3 (Пересылка), 7 (Досылка) | В доставке |
| 8 (Временное хранение), 12 (Прибытие в место вручения) | В пункте выдачи |
| 5 (Вручение) | Доставлен |
| 6 (Возврат) | Отменён |

### Connector config fields

- `login`, `password` — tracking.pochta.ru credentials
- `ordersTypeId` — EAV type ID of orders table
- `trackingReqId` — reqId where tracking number is stored
- `orderStatusReqId` — ref column for status
- `pvzFactReqId` — PVZ address field
- `deliveryMethodReqId` — ref column for delivery method
- `pochtaDeliveryId` — record ID for "Почта России" in delivery method lookup
- `terminalStatusIds` — status record IDs to skip (Доставлен, Завершён, Отменён)

### Rate limiting

Pochta API allows 100 requests/day for regular accounts (unlimited for postal contracts). `pollTracking` adds 200ms delay between requests when processing >10 orders.

## Enrichment Mode (Mode 3)

Set `enrichMode: true` in the connector config to switch from import mode (API → EAV) to enrichment mode (EAV → API → EAV). The connector reads existing rows from a source table, calls the external API once per row, and writes results back to the same row's fields.

```json
{
  "enrichMode": true,
  "sourceTypeId": 301,
  "url": "https://api.example.com/price/{{code}}",
  "method": "GET",
  "rowContext": {
    "code": { "reqId": 389, "regex": "[?&]CODE=(\\d+)" }
  },
  "skipIfMissing": ["code"],
  "mapping": { "data.0.value": 307 },
  "rateLimit": 500
}
```

### Config fields (enrichment mode only)

| Field | Type | Description |
|-------|------|-------------|
| `enrichMode` | boolean | Enables enrichment mode |
| `sourceTypeId` | int | Type ID of the table to enrich (reads all objects of this type, including child records) |
| `rowContext` | object | Maps URL template param names → EAV field config. Each entry: `{ reqId: N, regex?: "pattern" }`. If `regex` provided, extracts capture group 1 from the field value. |
| `skipIfMissing` | string[] | Skip rows where any of these param names resolved to null/empty |
| `rateLimit` | int (ms) | Delay between per-row API calls to avoid rate limiting |

### Built-in URL template variables (enrichment mode)

In addition to `rowContext`-derived params, the following variables are always available in the URL template:

| Variable | Value |
|----------|-------|
| `{{val}}` | URL-encoded display name of the source row (`encodeURIComponent(row.val)`) |
| `{{val_raw}}` | Raw display name (not encoded) |
| `{{id}}` | Object ID of the source row |

### Single-row enrichment via `_objectId`

When an automation triggers enrichment (e.g. `run_connector` action on `on_create`/`on_update`), it passes `_objectId: ctx.objId` to the connector. In this case, enrichment runs for **only that one row** instead of the entire source table. This makes automation-triggered enrichment efficient regardless of table size.

### Execution flow (enrichment)

1. If `_objectId` param provided → load only that row; otherwise load all root objects of `sourceTypeId` (ordered by id)
2. For each object: load its EAV fields, apply `rowContext` extraction to build URL template params
3. If any `skipIfMissing` param is empty → skip row
4. Interpolate URL with `rowContext` values + built-in variables (`{{val}}`, `{{id}}`), validate SSRF
5. Call external API, parse JSON response
6. Apply `mapping` paths to resolved values, upsert into EAV (UPDATE existing field or INSERT new)
7. Apply `rateLimit` delay between rows
8. Return `{ enriched, skipped, errors, total }` summary

**Note:** enrichment mode does NOT emit `bus.emit('object.created')` or `bus.emit('object.updated')` — automations will not trigger from enrichment runs.

**Not compatible with:** `targetTypeId`, `dataPath`, `writeMode`, `idField`, `pagination`, `incremental`.

## Relations (nested child import)

`config.relations[]` — массив вложенных маппингов для импорта child-записей вместе с родительскими.

Каждый relation:
```json
{
  "path": "items",
  "targetTypeId": 200,
  "parentReqId": 301,
  "nameField": "title",
  "mapping": [{ "source": "sku", "target": "Артикул" }],
  "skipDeleteIf": { "reqId": 150, "values": ["Оплачен", "Доставлен"] }
}
```

- `path` — **required** — dot-path to the child array in the API response item. Without this field the relation is silently skipped.
- `targetTypeId` — **required** — тип дочерней таблицы
- `parentReqId` — reqId ссылки на родителя в дочерней таблице
- `nameField` — поле из API-ответа для display name дочерней записи
- `mapping` — маппинг полей (аналогично основному)
- `skipDeleteIf` — не удалять/пересоздавать child-записи, если поле родителя (`reqId`) содержит одно из `values`
- `parseFn` — optional parser function name. When `rel.parseFn === 'parseOrderItems'`, the relation parses free-text composition fields into structured child items instead of reading from a JSON array path

## resolvePath features

### Static values
When a mapping path starts with `=`, the literal value after `=` is used instead of resolving from the API response. Example: `"=pending"` maps to the string `"pending"`.

### Array wildcard
`data.variants[].price` — резолвит в массив значений из всех элементов массива.

### Array coalesce
`["a.b", "c.d"]` — массив путей, берёт первый non-null результат.

### fn aggregation
Для array wildcard результатов:
```json
{ "reqId": 100, "fn": "min", "path": "data.prices[].value" }
```
Поддерживаемые функции: `min`, `max`, `sum`, `first`, `join`.

### Transforms
- `invert` — boolean negation (`true` → `false`)
- `epoch` — Unix timestamp → ISO date string

## Ref Resolution in Mapping

Mapping entries can include a `resolve` object to auto-resolve foreign keys during import:

```json
{
  "response.status": {
    "reqId": 150,
    "resolve": {
      "typeId": 50,
      "byReqId": 151,
      "valueMap": { "PAID": "Оплачен", "NEW": "Новый" },
      "create": true,
      "createNameField": "name"
    }
  }
}
```

| Field | Description |
|-------|-------------|
| `typeId` | Lookup table type ID to search in |
| `byReqId` | Column reqId to match against the API value |
| `valueMap` | Optional value mapping applied before lookup (API value → EAV value) |
| `create` | If `true`, auto-create a lookup record when no match found |
| `createNameField` | Field name for the auto-created record |
| `stripAfterColon` | If `true`, strips variant suffix from IDs before lookup (e.g. `"14233619:50 ml"` → `"14233619"`) |

Resolution results are cached per connector run for performance.

## Builtin Connector Types

`executeBuiltinConnector(connectorType, config, ctx)` — handles non-HTTP connector types called from automation `run_connector` action:

| Type | Description |
|------|-------------|
| `email` | Send email via SMTP |
| `http` | Single HTTP request (not paginated import) |
| `telegram` | Send Telegram message |
| `internal` | Internal platform operation |

These are triggered exclusively by automations, not via the REST `/connectors/:id/run` endpoint.

## Additional Config Fields

| Field | Type | Description |
|-------|------|-------------|
| `dataPath` | string | Dot-path to extract the items array from the API response (e.g. `"data.items"`) |
| `idField` | string | Field name in each API item used as the unique key for upsert (e.g. `"id"`, `"externalId"`) |
| `body` | object | Static JSON body to send with POST/PUT requests |
| `timeout` | int (ms) | Request timeout (default: 30000) |
| `csrfFlow` | boolean | Enable CSRF token preflight for SAP APIs |
| `skipExistingRecords` | boolean | When `true`, only create new records — skip updating existing ones entirely |

## AI Tools

### CRUD

| Tool | Risk Tier | Description |
|------|-----------|-------------|
| `list_connectors` | TIER_LOW | List all connectors in the workspace |
| `get_connector` | TIER_LOW | Get connector config by ID |
| `create_connector` | TIER_HIGH | Create a new connector |
| `update_connector` | TIER_HIGH | Update connector config |
| `delete_connector` | TIER_HIGH | Delete a connector |
| `run_connector` | TIER_HIGH | Execute a connector (import/enrich) |
| `list_connector_presets` | TIER_LOW | List available connector preset configs |

**Note:** `create_connector` and `update_connector` TOOL_DEFs accept a `type` param (connector type string) that the REST API does not accept — the tool-executor maps it into `config` internally.

### Wizard

| Tool | Risk Tier | Description |
|------|-----------|-------------|
| `fetch_api_docs` | TIER_LOW | Fetch and parse external API documentation from a URL |
| `generate_connector_config` | TIER_MEDIUM | Generate a connector config from API docs and user requirements |
| `test_connector_draft` | TIER_HIGH | Test a draft connector config by running a limited fetch |
| `generate_connector_schema` | TIER_HIGH | Generate a target table schema from connector sample data |
| `reconcile_cdek` | TIER_HIGH | Reconcile CDEK shipment statuses |
| `discover_connector_schema` | TIER_LOW | Discover OData schema from connector URL |
| `search_prices` | TIER_LOW | Search prices via browser scraper |

## SSRF Protection

All connector URLs are validated through `utils/url-guard.js` before execution — blocks private IP ranges and localhost.

## DB Tables

- `_v2_connectors` (per-workspace) — connector definitions with JSONB config
- `_v2_connector_state` (per-workspace) — incremental sync cursors per connector
