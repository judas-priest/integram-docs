# Module: portal

**Path:** `src/api/v2/modules/portal/`
**Files:** `router.js`, `service.js`, `auth.js`, `api.js`, `private-api.js`, `write-api.js`, `analytics.js`, `cart.js`, `guest-order.js`, `legal.js`, `config-utils.js`, `roles.js`, `a2a.js`, `agent.js`, `chat/`, `teamchat/`, `meta-kb-proxy.js`, `decisions-proxy.js`
**Base URL:** Various — see below
**Auth:** Mixed — see per-endpoint sections.

## Purpose

Customer-facing portal. Each workspace can expose a public website powered by Nuxt SSR (port 3000). The portal serves a catalog, shopping cart, orders, client profile, knowledge base, AI chat, and custom pages. Portal users authenticate via OTP (SMS or Telegram bot). Role-based page/module visibility.

## URL Structure

| Pattern | Description |
|---------|-------------|
| `GET /portal/check-domain` | Global — for Caddy domain routing |
| `GET|POST /:db/portal/api/*` | JSON API endpoints (public or portal JWT) |
| `GET|POST /:db/portal/auth/*` | Auth endpoints (OTP, Telegram) |
| `GET /:db/portal/*` | Proxy to Nuxt SSR (port 3000) |

## Auth Endpoints (inline in `router.js`, helpers in `auth.js`)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/portal/api/auth/request` | Request OTP via SMS. Body: `{ phone }` |
| POST | `/portal/api/auth/verify` | Verify OTP, set `portal_jwt` httpOnly cookie. Body: `{ phone, code }` |
| POST | `/portal/api/auth/telegram/start` | Initiate Telegram bot login (returns `sessionToken` + deep link) |
| GET | `/portal/api/auth/telegram/status?token=X` | Poll for Telegram login result |
| GET | `/portal/api/auth/me` | Return current client info from JWT cookie |
| POST | `/portal/api/auth/logout` | Clear `portal_jwt` cookie |
| POST | `/portal/api/auth/staff/login` | Email+password login for workspace staff members. Issues portal_jwt with `user_type:'portal_staff'`, 30-day cookie. |

Telegram webhook is registered globally at `POST /portal/telegram/:secret` (no `:db` prefix, handled by `globalRouter`). Handled update types: `message` (deep link auth, contact share, `successful_payment`), `callback_query` (inline keyboard button presses from automations), `edited_message`, `my_chat_member`, `inline_query`, `pre_checkout_query`, `shipping_query`, `chat_join_request`, `business_connection`, `business_message`.

### Telegram Bot Auth Flow

1. User clicks "Login via Telegram" → `POST /portal/api/auth/telegram/start`
2. Backend resolves `botToken` + `botUsername` — first from portal config `notifications`, then falls back to first enabled bot in `_v2_tg_bots`
3. Generates session token, stores in Redis (TTL 5 min)
4. Returns deep link: `https://t.me/{BOT_USERNAME}?start={token}`
5. User opens Telegram → bot requests phone contact (or auto-authorizes if chat_id known)
6. Bot webhook fires → backend finds/creates client, marks session `authorized`
7. Frontend polls `GET /portal/api/auth/telegram/status` → receives JWT cookie

### Telegram Inline Keyboard Callbacks

Automations send Telegram messages with inline keyboard buttons via `send_telegram` action. Callback dispatch lives in `portal/telegram-callback.js`.

Supported `callback_data` ops:

**Navigation**

| op | meaning |
|----|---------|
| `n` | Navigate to a named screen |
| `b` | Jump back to a named screen |

**Field updates**

| op | meaning |
|----|---------|
| `u` | Set EAV field (`objId:reqId:value`); clears the keyboard on success |
| `t` | Toggle bool (`objId:reqId`): `'1'`↔`'0'`; flips the leading icon (⬜↔✅) of the matching button only |
| `c` | Set parent reqId=value, but only if every child has the toggle ≡ true (`parentId:reqId:value:childTypeId:toggleReqId`) |
| `mu` | Multi-field update — set several EAV fields atomically |
| `ef` | Edit field conversationally (prompts user for free-text input) |
| `di` | Dimensions input (structured W×H×D entry) |

**Status change (NEW)**

| op | meaning |
|----|---------|
| `sc` | Open status picker — shows available status transitions |
| `scc` | Status change confirmation — display confirm/cancel keyboard |
| `scx` | Status execute — apply the chosen status transition |

**Assembly**

| op | meaning |
|----|---------|
| `at` | Toggle assembly item collected/not-collected |
| `ab` | Batch mark all items as collected |

**CDEK**

| op | meaning |
|----|---------|
| `cs` | Initiate CDEK shipment creation |
| `pl` | Print CDEK label |
| `cbc` | Batch confirm CDEK shipments |
| `nos` | Set new order source |

**Stock**

| op | meaning |
|----|---------|
| `sa` | Adjust stock quantity |
| `si` | Quantity input (prompts for number) |
| `sn` | Create new stock record |

**Order linking**

| op | meaning |
|----|---------|
| `los` | Link-order search — find orders to link |
| `lo` | Confirm order link |
| `lou` | Unlink order |

**Search**

| op | meaning |
|----|---------|
| `os` | Order search |
| `cls` | Client search |

**Other**

| op | meaning |
|----|---------|
| `ap` | Action placeholder / cancel (no-op button) |
| `cco` | Client/custom order ops |
| `cca` | Client/custom order accept |

Modifiers (joined to the args with `;`):

- `r=<roleObjId>[,<roleObjId>…]` — sender's employee role must equal one of the listed ids (comma-separated, no spaces required). Use for buttons that should be available to multiple roles, e.g. `r=419,420` lets both пчеловод and сборщик click. A single id keeps the legacy syntax.
- `f=<expectedCurrentValue>` — CAS guard
- `l=<reqId>:<maxVal>` — refuse when `obj[reqId] >= maxVal` (numeric)
- `lp=<reqId>:<maxVal>` — same check but on the parent record (loaded via `up`). Use for child-table buttons that should be guarded by the parent's status (e.g. cancel a Дозаказ only while the parent Заказ is below «В доставке»). Returns «Нет родительской записи» when the object has no parent (`up=1`).
- `T=<reqId>` — write sender's employee record-id to this field (only if currently empty)

Flow on click:

1. Telegram sends `callback_query` to `POST /portal/telegram/:secret`
2. Workspace resolved via `_v2_portal_config.telegram_secret`
3. `handleTelegramCallback()` parses, resolves the sender employee (using `config.notifications.telegramEmployeeTable`), enforces modifiers, dispatches by op
4. `answerCallbackQuery` removes the loading spinner with a short status reply
5. Keyboard is cleared (`u`/`c`) or rebuilt with one flipped icon (`t`) on the clicked message
6. **Broadcast sync** — if the message was part of a multi-recipient broadcast (`recipients.fromTable`), the same keyboard transition is fanned out to every other recipient's message of the same trigger object. See `### Broadcast keyboard sync` below.
7. `bus.emit('object.updated', …)` so downstream automations / webhooks / graph react

**Sender resolution requires** `portal_config.notifications.telegramEmployeeTable` (or `botConfig.employeeTable` — takes precedence when the callback arrives via a named bot config rather than the default portal bot):

```json
{
  "typeId": 313,
  "chatIdReqId": 354,
  "roleReqId": 380
}
```

Without it, `requireRole` is unenforceable and clicks with `r=` modifier are denied. Setting it once per workspace is enough — every automation reuses the same config.

**Key file:** `backend/src/api/v2/modules/portal/telegram-callback.js`
**Helpers (auth.js):** `telegramAnswerCallbackQuery`, `telegramEditMessageReplyMarkup`
**Broadcast helpers (telegram-broadcast.js):** `ensureSentMessagesTable`, `recordSentMessage`, `loadOtherSentMessages`, `deleteSentMessagesForObject`, `updateSentMessageMarkup`
**Service (service.js):** `resolveByTelegramSecret`, `resolvePortalBotToken`
**DB column:** `_v2_portal_config.telegram_secret` (stored on webhook registration)
**Per-workspace satellite:** `_v2_tg_sent_messages` (recipients of every broadcast — chat_id, message_id, reply_markup, obj_id). Ensured by `ensureAutomationsTable`.

### Broadcast keyboard sync

When `send_telegram` action broadcasts via `recipients.fromTable`, each active recipient gets their own message with a unique `(chat_id, message_id)`. Without sync, after one recipient clicks the keyboard updates only on that one message — the others stay clickable indefinitely. Stale clicks either spam «Уже обработано» (best case) or lose the CAS race (`fromValue` guard catches the duplicate update).

**Implementation:**

1. After every successful `executeBuiltinConnector('telegram', …)` call inside `send_telegram` action, the resulting `message_id` plus the per-recipient `reply_markup` are persisted in `_v2_tg_sent_messages (automation_id, obj_id, chat_id, message_id, reply_markup, sent_at)`. Only rows with an inline keyboard are persisted — plain text messages never need a sync.
2. On a successful `u` or `c` op (keyboard cleared on success), the dispatcher loads every other row for `obj_id`, calls `editMessageReplyMarkup` with an empty keyboard on each, then deletes all rows for `obj_id` since the keyboard is gone everywhere.
3. On a `t` op (one icon flipped, keyboard stays), the dispatcher applies `buildToggleFlippedKeyboard` to each sibling's stored markup, edits the message, and updates `reply_markup` in the satellite for next iteration.
4. Per-sibling edit failures (`message not found`, blocked bot, deleted chat) are logged and skipped — the loop continues for the rest.

**Authority.** The EAV record is the source of truth. The sync is a best-effort visual catch-up — if `editMessageReplyMarkup` fails for some recipient, the worst that happens is they keep a stale-looking keyboard locally; clicks still hit CAS/role guards on the backend.

**Trade-offs.**

- **Concurrent clicks.** Two recipients can race on the same button before sync edits arrive. First click wins by CAS; the second gets «Статус уже изменился». The losing recipient's keyboard is then edited by the first sync pass, so a third click from them isn't possible.
- **Telegram rate limits.** `telegramEditMessageReplyMarkup` goes through the same global throttle (30/sec, 1/sec per chat). Broadcast sizes < 30 fan out within one second; larger broadcasts are queued.
- **Cleanup.** `_v2_tg_sent_messages` rows for an object are deleted on the first successful `u`/`c` click. If the broadcast never gets a click (e.g. all recipients abandon it), rows survive until manually pruned; consider adding a TTL job if accumulation becomes an issue.

### Telegram webhook hardening

Three layers protect the `/portal/telegram/:secret` endpoint:

1. **Header verification** (`telegram-webhook-secret.js`) — every request must carry
   `X-Telegram-Bot-Api-Secret-Token` matching the URL `:secret` (constant-time compare).
   Rejected with 401 otherwise. The header is set by `setWebhook` via the
   `secret_token` parameter (see `dev-tunnel.js`, `auth.js#telegramSetWebhook`).

2. **Update_id deduplication** (`telegram-dedup.js`) — every `update.update_id` is
   stored in Redis under `tg_update_seen:<botSecret>:<updateId>` with 24h TTL via
   `SET NX`. Duplicate deliveries are skipped. Fail-open if Redis is down or the
   SET call rejects.

3. **Outbound rate limiting** (`telegram-rate-limit.js`) — `telegramSendMessage`
   wraps every call in a token-bucket throttle: 30/sec global, 1/sec per chat.
   On a 429 response from Telegram, the wrapper sleeps `retry_after` seconds and
   retries once. Disable with `TELEGRAM_THROTTLE=0` (tests only).

### Telegram Bot Management

See dedicated module documentation: **[telegram-bots.md](telegram-bots.md)** — multi-bot constructor, config schema, API endpoints, automation integration, migration guide.

### portalAuth(...roles) Middleware

Unified auth middleware for portal endpoints. Replaces ad-hoc `requirePortalJwt` + role check patterns. Returns an array of 3 middleware functions: `[parseJwt, resolveGrants, checkRole]`.

**Usage:**

```js
router.get('/endpoint', ...portalAuth('admin', 'editor'), handler);
```

| Call | Behavior |
|------|----------|
| `portalAuth('admin', 'editor')` | Requires portal JWT + role must be `admin` or `editor` |
| `portalAuth('*')` | Requires portal JWT, any role accepted |
| `portalAuth()` | Optional auth — parses JWT if present, does not reject anonymous |

**Flow:** Parse `portal_jwt` cookie -> resolve grants via `resolvePortalUserGrants()` -> check role against allowed list.

**File:** `auth.js` — `portalAuth`

### Portal JWT

Separate from workspace JWT. Signed with `PORTAL_JWT_SECRET`. Payload (`signPortalJwt`): `{ db, clientObjectId, user_type: 'portal_customer' }`. Staff tokens (`signStaffPortalJwt`) carry `{ db, userId, email, user_type: 'portal_staff' }` plus optional `clientObjectId`. Checked via `requirePortalJwt` / `optionalPortalJwt` (legacy) or `portalAuth(...roles)` (preferred).

**Capability-URL alias:** `PORTAL_ALIAS_DB` (constant in `nuxt.config.ts`, mirrored in `portal/composables/portalAlias.js`) — a workspace can be entered by a short alias token without `db`/`portal` in the URL: `/<TOKEN>` and `/<TOKEN>/auth`. The token pins the workspace, so alias pages carry no `db` segment — all readers of `route.params.db` must go through `resolvePortalDb()`; canonical `/:db/portal/...` paths resolve unchanged.

## Public API Endpoints (`api.js` + inline in `router.js`)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/portal/api/config` | Portal config (pages, modules, theme, connectors flags). **Secrets stripped** by the platform-wide `stripSecretValues` (`sanitizePublicConfig`, issue #143). No auth: the same response is embedded in the portal SSR HTML |
| GET | `/portal/api/catalog` | Product/service catalog (`?page=`, `?limit=`, `?category=`) |
| GET | `/portal/api/catalog/categories` | Category list for filter UI (from `categoryTypeId`) |
| GET | `/portal/api/catalog/:slug` | Single catalog item by slug |
| GET | `/portal/api/sitemap.xml` | XML sitemap — includes catalog items + all public portal pages from config (with `lastmod`, `changefreq`, `priority`). Excludes protected pages (auth, cart, profile, orders, documents, metrics, support, wishlist) |
| GET | `/portal/api/gallery` | Public image files list (`?folder=`) |
| GET | `/portal/api/files/:filename` | File download, `optionalPortalJwt`. Images (`.jpg/.jpeg/.png/.gif/.webp/.svg`) are open to anyone — portal branding must render before login. Audio/video (`MEDIA_EXTS`: `.webm/.ogg/.oga/.opus/.m4a/.mp3/.wav/.mp4/.mov`) require a portal session — 401 `AUTH_REQUIRED` without it. Any other extension — 403 `FORBIDDEN` |
| GET | `/portal/api/reports/:reportId` | Report data for portal widgets. Only reports referenced in portal config are allowed (403 otherwise — `isReportReferencedInConfig` check) |
| GET | `/portal/api/kit/catalog` | @kit component catalog for the section editor (`?version=`, `?kind=`, `?search=`; `?name=` returns full details of one component). `optionalPortalJwt`. `Cache-Control: public, max-age=300`. Error codes: 503 `KIT_NOT_DEPLOYED` (no deployed library), 404 `KIT_VERSION_NOT_FOUND` / `KIT_CATALOG_MISSING` |
| GET | `/portal/api/kit/versions` | Deployed @kit versions (version Select in custom_code settings). Same auth and status map as the catalog route |
| GET | `/portal/robots.txt` | SEO robots.txt |
| GET | `/portal/api/config/full` | Raw config **including secrets** (admin JWT). The visual portal editor loads its config from here — the public route strips secrets and saving back what it read would wipe them |
| POST | `/portal/api/config` | Create/update portal config (admin JWT). Runs `validatePortalRefs` — 400 `INVALID_CONFIG_REFS` on newly introduced broken refs |
| GET | `/portal/api/config/history` | Last config snapshots (admin JWT, `?limit=` 1–100, default 20). Rows carry secrets, same exposure as `/api/config/full` |
| POST | `/portal/api/config/restore` | Restore config from a snapshot (admin JWT). Body: `{ historyId }`. Skips ref validation by design |
| PUT | `/portal/api/config/active` | Toggle portal active flag (admin JWT) |

## Private API (portal JWT required, `private-api.js`)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/portal/api/profile` | Client profile |
| PUT | `/portal/api/profile` | Update profile |
| GET | `/portal/api/orders` | Client's orders list |
| GET | `/portal/api/orders/:orderId` | Order detail |
| GET | `/portal/api/documents` | Client documents (`?docType=`, `?docStatus=`, `?from=`, `?to=`) |
| GET | `/portal/api/metrics` | Client metrics/KPIs |

## Read API for Custom Code (portal JWT required, inline in `router.js`)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/portal/api/tables/:typeId/objects` | List EAV records of a type (`optionalPortalJwt` — unauthenticated allowed for config-referenced types). Params: `?search=`, `?limit=`, `?offset=`. Rate-limited: 200/min |
| GET | `/portal/api/documents/:docId` | Get document with blocks (portal JWT, ownership-checked — only docs belonging to the authenticated client). Rate-limited: 60/min. Staff branch (workspace staff JWT) is guarded by `portalGrantsMiddleware` and runs the workspace lazy migrations (`ensureSideTables(pool, db, 'documents')`) before querying. `PATCH /api/documents/:docId` changes only the document title |
| GET | `/portal/api/timeseries/:sourceId` | Query timeseries data. Params: `?metric=`, `?from=`, `?to=`, `?bucket=`, `?agg=`, `?limit=`. Rate-limited: 30/min |
| GET | `/portal/api/graph/:objId/related` | Get graph neighbors. Params: `?direction=in|out|both`, `?limit=`. Rate-limited: 30/min |
| POST | `/portal/api/upload` | File upload from custom_code (portal JWT, 20 req/min, max 50 MB). Returns `{ id, filename, url }` |

## Write API for Custom Code (`write-api.js`)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/portal/api/objects` | Create EAV record (portal JWT + WRITE grant for typeId). Rate-limited: 30/min |
| PATCH | `/portal/api/objects/:id` | Update EAV record fields (portal JWT + WRITE grant). Rate-limited: 60/min |
| POST | `/portal/api/connectors/:id/run` | Run connector by ID (portal JWT + READ grant). Rate-limited: 10/min |
| DELETE | `/portal/api/objects/:id` | Delete a portal object (calls portalDeleteObject). Portal JWT + WRITE grant for typeId. |
| GET | `/portal/api/codespace/:repo/blob/:ref/*` | Read codespace file for custom_code modules, `optionalPortalJwt`. Four gates in order: repo must be referenced by an active custom_code module (403 `REPO_NOT_IN_CONFIG`); `ref` must be one the config names (403 `REF_NOT_ALLOWED`); if the module is not anonymous — portal session required (401 `AUTH_REQUIRED`) and the client role must see that module (403 `ROLE_REQUIRED`, role via `loadClientRole`); file extension must be on the whitelist (403 `FILE_TYPE_NOT_ALLOWED`, `isBlobFileAllowed` in `code-guard.js`). ETag = commit sha, 304 on match |

## Staff API

Endpoints for authenticated workspace staff members. All require `portal_jwt` with `user_type:'portal_staff'`.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/portal/api/staff/me` | Get employee info for the authenticated staff member. Requires `portalAuth('admin', 'owner', 'editor')` |
| POST | `/portal/api/staff/items/:itemId/toggle` | Toggle Собрано (collected) flag on an order item |
| POST | `/portal/api/staff/orders/:id/check-all-collected` | Check whether all items in an order are collected |
| POST | `/portal/api/staff/orders/:id/items` | Add a product to an order |
| GET | `/portal/api/staff/orders/:id/linked` | Find orders linked to this order |
| POST | `/portal/api/staff/orders/:id/collect-all` | Mark all items in an order as collected |
| POST | `/portal/api/staff/orders/merge` | Merge donor orders into a master order |
| GET | `/portal/api/staff/events` | SSE stream for real-time events (order updates, notifications) |
| GET | `/portal/api/staff/cdek/config` | Get CDEK connector configuration for the workspace |
| POST | `/portal/api/staff/orders/:id/cdek/calculate` | Calculate CDEK delivery tariff for an order |
| GET | `/portal/api/staff/cdek/pvz` | Search CDEK pickup points (PVZ) |
| POST | `/portal/api/staff/dadata/suggest-address` | DaData address autocomplete |
| POST | `/portal/api/staff/dadata/suggest/:type` | Generic DaData suggestions proxy (address, party, bank, fio, email). Body: `{ query, count? }` |
| POST | `/portal/api/staff/dadata/findById/:type` | DaData findById proxy (party by INN, address, bank). Body: `{ query }` |
| POST | `/portal/api/staff/orders/:id/cdek/shipment` | Create CDEK shipment for an order |
| GET | `/portal/api/staff/orders/:id/cdek/label` | Download CDEK label PDF |
| GET | `/portal/api/staff/product-variants` | All product variants grouped by product |

### POST /api/agents/invoke-many

Параллельный fan-out к нескольким внешним агентам. Требует portal JWT.

**Request body:**
```json
{
  "slugs": ["design-agent", "quality-agent"],
  "task": "Analyze part ECO-123",
  "context": { "partId": 456 }
}
```

**Response:**
```json
{
  "ok": true,
  "data": {
    "results": [
      { "slug": "design-agent", "ok": true, "result": {} },
      { "slug": "quality-agent", "error": true, "message": "timeout" }
    ],
    "traceId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

Ограничения: max 10 агентов; требует `agent.enabled: true` в portal config. Graceful degradation: ошибка одного агента не прерывает остальных.

All endpoints require `portalAuth` (or legacy `requirePortalJwt`) unless noted. Access controlled by `_v2_grants` via `resolvePortalUserGrants()`, with fallback to config-reference check when no grants are configured. Object writes are logged with `portalClientId` and trigger `object.created`/`object.updated` events for automations. Connector run delegates to `executeConnector` from `connectors/service.js` with a synthetic user that passes the internal grant check.

## Cart (`cart.js`)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/portal/api/cart` | Get cart items |
| POST | `/portal/api/cart/items` | Add item |
| PUT | `/portal/api/cart/items/:itemId` | Update quantity |
| DELETE | `/portal/api/cart/items/:itemId` | Remove item |
| DELETE | `/portal/api/cart` | Clear cart |
| POST | `/portal/api/cart/merge` | Merge guest cart into authenticated cart |

## Orders (`guest-order.js`, `router.js`)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/portal/api/orders/guest` | None | Place order without auth (name, phone, items). Finds or creates client by phone. |
| POST | `/portal/api/orders` | portal_jwt | Place order as authenticated portal user. Uses `clientObjectId` from JWT — no phone lookup. |

Both endpoints support optional `ordersConfig` fields:
- `sourceReqId` + `portalSourceId` — writes source (e.g. "Портал") to order on creation

### Catalog inStock filter (`api.js`)

If `inStockReqId` is set in catalog config, `getCatalog` adds a `HAVING` clause to exclude items where the in-stock field is `'0'`, `'false'`, `''`, or `'нет'`.

## Analytics (`analytics.js`)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/portal/api/analytics/event` | Record analytics event (public, fire-and-forget). Events: `page_view`, `product_view`, `order_created` |
| GET | `/portal/api/analytics/stats` | Aggregate usage statistics (admin only, `?period=1d\|7d\|30d`) |

## AI Chat (`chat/`)

AI chat widget for portal visitors. Sub-router at `/portal/api/chat/`. Conversations with context from the workspace knowledge base. See `portal/chat/router.js` for endpoint details.

## Support Tickets (portal JWT required, inline in `router.js`)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/portal/api/support/tickets` | List client's support tickets |
| POST | `/portal/api/support/tickets` | Create new ticket. Body: `{ subject, message }` |
| GET | `/portal/api/support/tickets/:ticketId` | Get ticket detail |

## Knowledge Base (public, inline in `router.js`)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/portal/api/kb/articles` | List KB articles (`?category=` filter) |
| GET | `/portal/api/kb/articles/:articleId` | Get KB article detail |
| GET | `/portal/api/kb/search?q=` | Semantic search over KB articles (rate-limited) |

## CDEK Delivery (inline in `router.js`)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/portal/api/delivery/calculate` | Calculate delivery cost via CDEK API (public, rate-limited) |
| POST | `/portal/api/cdek/webhook/:secret` | CDEK status webhook receiver (no auth) |
| POST | `/portal/api/uds/webhook/:secret` | UDS loyalty system webhook handler (no auth) |

## Legal Docs (`legal.js`)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/portal/api/legal/generate` | AI-generated legal page drafts (admin only) |

## Portal Config (`config-utils.js`)

Portal configuration stored in `_v2_portal_config` (global schema). Config includes:
- `theme`: colors, fonts, logo, favicon
- `pages`: array of page definitions (`{ type, slug, title, modules }`)
- `modules`: enabled feature modules per page
- `chat`: AI chat settings

`KNOWN_PAGE_TYPES` and `KNOWN_MODULE_TYPES` defined in `config-utils.js`.

### Secrets in config

The portal does **not** own a secret classifier. `sanitizePublicConfig()` is the platform's
`stripSecretValues()` from the workspace-carry registry (`src/api/v2/registry/workspace-carry.js`) —
the same function that strips secrets when a workspace is cloned, exported as a template, or dumped
into a diagnostics snapshot. A second pattern next to it would drift silently and answer differently
for the same key depending on which door the data left through.

The rule judges the **key name**, not the value type: `SECRET_KEY_RE` is
`/secret|token|api_?key|password|authorization|bearer/i`, and a number under such a name is dropped
just like a string. On top of the pattern the portal binds the three **pointer keys** the same
registry declares for `_v2_portal_config.config` — `telegramTokenSource`, `telegramIntegrationId`,
`telegramTokenReqId`. They are the second road to the same bot token (`resolvePortalBotToken`), and
nothing outside the server reads them: a full sweep of `portal/` and `frontend/src` finds no reader.
The list is taken *from* the registry, never re-typed here.

Today the rule matches `notifications.telegramBotToken` plus those pointers, and nothing else —
measured over 84 live configs on the dev database (22.08.2026).

Two readers, two answers:

- `GET /portal/api/config` — public, secrets removed by `sanitizePublicConfig()` (arrays are walked
  too). This response also ends up inside the portal SSR HTML, so anything left in it is world-readable.
- `GET /portal/api/config/full` — admin JWT, raw row. The portal editor reads this one.

AI tools follow the same split: `get_portal_config`, `set_portal_config` and `update_portal_module`
sanitize the config before returning it (the last two used to return the whole `_v2_portal_config`
row from `RETURNING *`). The database always stores the full config — sanitizing happens on the response only.

The **`telegram_secret` column** needs no sanitizer of its own, because no query hands it out any
more. It is `sha256(bot token)` truncated to 32 chars — both the webhook path
(`/api/v2/portal/telegram/:secret`) and the value Telegram signs incoming updates with, so anyone
holding it can forge "messages from Telegram". `getConfig` never selected it, but `upsertConfig`
and `setActive` used `RETURNING *`, so it rode out through `POST /api/config`,
`PUT /api/config/active` and the two writing AI tools. Both now name their columns explicitly, from
the single `PUBLIC_ROW_COLUMNS` list shared with `getConfig` — read and write cannot drift apart,
and a column added to the table is not published by accident. Only `resolveByTelegramSecret` reads
it, by name, in its own query. Guard: `__tests__/upsert-config-guards.test.js` compares the whole
`RETURNING` list, not just the absence of the name — `RETURNING *` contains no name either and
would pass an "is the name absent" check.

### Write contract (`upsertConfig`)

- **`config` field absent from the request → the `config` column is not touched at all.** Before
  issue #143 a `POST /portal/api/config` carrying only `custom_domain` wrote `'{}'` over the entire
  config — saving a domain from the workspace card erased the portal.
- **A secret the incoming config does not name is carried over** from the stored config
  (`preserveSecrets`), so writing back a config that was read from the public route does not lose
  the bot token. What is carried over is exactly what `sanitizePublicConfig` removes — both ask the
  same `isSecretKey`, pointers included. Were the two to disagree, a client that read the public
  config and wrote it back would lose precisely what was missing from what it read. A whole section
  missing from the new config is carried over **in full** if it contains a secret.
- **Deleting a secret requires naming it explicitly** as `null` or `''`. Absence is never deletion.
- **Arrays are not walked by `preserveSecrets`** — pages have no stable match by position, and
  carrying over by index would graft a secret onto the wrong page. A secret nested inside `pages[]`
  is therefore stripped on read but not restored on write.

Guards: `src/api/v2/registry/__tests__/carry-invariants.test.js` (one classifier, not two),
`__tests__/upsert-config-guards.test.js` (write contract),
`__tests__/public-config-secrets.test.js` and `__tests__/config-secrets.test.js` (public response),
`ai/agent/tools/__tests__/portal-config-secrets.test.js` (AI tools).
`backend/scripts/portal-secret-audit.mjs` lists configs that still hold secrets and exits 1 while
any remain.

### Config history and ref validation (incident 2026-09-03)

Incident: a config pointing at another workspace's repo and tables was accepted silently and
destroyed the workspace's live config. Two mechanisms came out of it:

**History.** Any write that carries a `config` field (`upsertConfig`) first snapshots the
PREVIOUS config into `_v2_portal_config_history (id, db, config JSONB, changed_by, changed_at)`
— the only rollback mechanism the config has, no audit log otherwise. The insert runs under a
`SAVEPOINT` inside the same transaction as the upsert: a failed snapshot degrades to a log error
("no rollback for this edit"), never breaks the save. `changed_by` is the REST user's email
(`req.v2user.email`); MCP write paths do not pass an actor yet, so rows read `'unknown'`.
Read/restore: `GET /api/config/history` (`?limit=` 1–100, default 20) and
`POST /api/config/restore { historyId }`, both admin-only. Restore goes through `upsertConfig`,
so the replaced config lands in history itself (undo of undo loses nothing) — and **deliberately
skips ref validation**: a legacy config must be restorable even if it would fail today's checks.

**Ref validation** (`validatePortalRefs` in `config-utils.js`) complements the structural
`validatePortalConfig` — that one asks "is it well-formed", this one "does it point at anything":
`custom_code` `repo` must exist in the workspace (`REPO_NOT_IN_WORKSPACE`), `file` must be in the
repo root at the named ref (`FILE_NOT_IN_REPO`), `bindings` values `table:N` must be existing
tables (`TABLE_NOT_IN_WORKSPACE`). A judge failure (e.g. git down) is not a silent skip — it
becomes `CHECK_FAILED`. Enabled on all three writers: REST `POST /api/config` (400
`INVALID_CONFIG_REFS`), MCP `set_portal_config` and `update_portal_module` (the latter two had no
validation at all before). Defects already present in the stored config do not block the edit —
error budget counted against the previous config, same rule as `validatePortalConfig`.

Table lifecycle: in the workspace-carry registry with `clone/template: skip`; workspace purge
finds its rows via the `db` column like `_v2_portal_config` itself. Guards:
`__tests__/config-history.test.js`, `__tests__/config-refs-validate.test.js`.

## Roles (`roles.js`)

Portal access control uses `_v2_grants` via `resolvePortalUserGrants()` (from `utils/v2-grants.js`) and `portalGrantsMiddleware`. The middleware resolves the portal client's workspace role and grants, attaching them to `req.portalGrants`. Write API and read API endpoints use `checkGrant()` against these resolved grants to enforce per-typeId READ/WRITE permissions. Role-based page/module filtering:
- `loadClientRole(pool, db, authConfig, clientObjectId)` — loads role from EAV
- `filterPagesByRole(pages, role)` — removes pages not visible to role
- `filterModulesByRole(modules, role)` — removes modules not visible to role

## Nuxt SSR Proxy

All non-API portal routes proxy to `http://127.0.0.1:3000` (Nuxt SSR). The Nuxt app calls the `/portal/api/*` endpoints for data.

## Domain Routing

`GET /portal/check-domain?domain=example.com` — called by Caddy to determine which workspace handles a custom domain. Returns `db_name` if domain is configured.

## DB Tables

- `_v2_portal_config` (global) — portal configuration per workspace
- `_v2_portal_config_history` (global) — snapshot of the previous config before every write; source for `GET /api/config/history` and `POST /api/config/restore`
- `_v2_portal_events` (global) — analytics events
- `_v2_portal_clients` (per-workspace, lazy-init) — portal user accounts
- `_v2_portal_cart` (per-workspace, lazy-init) — shopping cart items
- `_v2_portal_otp` (per-workspace, lazy-init) — OTP codes with TTL
- `_v2_grants` (per-workspace) — role-based access grants for portal users (READ/WRITE per typeId)
- Redis — Telegram session state (TTL 5 min)

## Agent API

### POST /:db/portal/api/agent/run

Requires: `portal_jwt` cookie (customer or staff).

Request body: `{ "message": "string (required)", "threadId": "string (optional)", "agentSlug": "string (optional)", "searchMode": "string (optional)", "loopMode": "boolean (optional)", "enabledAgents": "string[] (optional)" }`

Response: `text/event-stream` — SSE events:
- `{ "type": "RUN_STARTED" }`
- `{ "type": "TEXT_MESSAGE_CONTENT", "content": "..." }`
- `{ "type": "TOOL_CALL_START", "toolName": "...", "args": {...} }`
- `{ "type": "TOOL_CALL_END", "toolName": "...", "result": {...} }`
- `{ "type": "STATE_DELTA", "delta": [...] }` — JSON Patch ops on `/memories/{key}` or `/shared/{key}` paths
- `{ "type": "RUN_FINISHED" }`
- `{ "type": "error", "message": "..." }`

Returns 403 if `portal.config.agent.enabled` is not `true`.

### POST /:db/portal/api/agent/resume

HITL (human-in-the-loop) approval endpoint. Resumes an agent run after a tool call requires user confirmation.

Requires: `portal_jwt` cookie.

Request body: `{ "threadId": "string (required)", "approved": "boolean (required)" }`

### GET /:db/portal/api/meta-kb/workspace-tools

List active workspace tools (for tool toggles UI). Returns `{ ok: true, data: [...] }`.

Requires: `portal_jwt` cookie + `metaKb.enabled` config flag.

### PATCH /:db/portal/api/meta-kb/agents/:id/tools

Update agent tool toggles. Portal can only modify internal agents' tool lists.

Requires: `portal_jwt` cookie + `metaKb.enabled` config flag.

### POST /:db/portal/api/a2a

Receives incoming A2A (Agent-to-Agent) JSON-RPC tasks from peer agents.
Requires `Authorization: Bearer <portal.config.agent.apiKey>`.

Accepted methods: `message/send` (A2A ≥ v0.2.0), `tasks/send` (pre-0.2 alias), `tasks/get`.
`message/stream` is not accepted — the endpoint returns a single JSON-RPC response and
its Agent Card declares `capabilities.streaming = false`.

Request format: A2A JSON-RPC `message/send` message
```json
{
  "jsonrpc": "2.0",
  "method": "message/send",
  "params": {
    "id": "task-uuid",
    "message": { "role": "user", "parts": [{ "kind": "text", "text": "task description" }] }
  }
}
```

`params.id` is optional for spec-current callers: the task id is taken from
`params.message.taskId` when resuming an existing task, and assigned by the server
when neither is present.

Response: A2A JSON-RPC result with artifacts
```json
{
  "jsonrpc": "2.0",
  "result": {
    "id": "task-uuid",
    "message": { "role": "assistant", "parts": [...] },
    "artifacts": [...]
  }
}
```

Returns 401 if the Bearer API key is missing or wrong, 403 if no API key is configured
or `portal.config.agent.enabled` is not `true`.
If the orchestrator requires elicitation or HITL (human-in-the-loop) confirmation, the
response is HTTP 200 with task state `input-required` (see ADR-009), not 409.

### Swarm Memory Integration (Chat)

When portal chat (`/portal/api/chat/message`) or orchestrator (`/portal/api/a2a`) runs, swarm memory is integrated:
- **Before LLM**: `recall()` injects relevant memories into system prompt
- **After run**: `extractSessionInsights()` fire-and-forget — analyzes conversation, extracts insights, stores to swarm memory

STATE_DELTA events include JSON Patch operations on:
- `/memories/{key}` — assistant memory store (structured facts, insights, observations)
- `/shared/{key}` — cross-session shared state (aggregated metrics, patterns)

### Portal config — agent section

```json
{
  "agent": {
    "enabled": true,
    "tools": {
      "allowWrite": false,
      "allowDelete": false,
      "allowSchema": false
    },
    "roleOverrides": {
      "manager": { "allowWrite": true },
      "admin":   { "allowWrite": true, "allowDelete": true, "allowSchema": true }
    }
  }
}
```

`tools` = base permissions for all portal users.
`roleOverrides` = per-role overrides matched by `roleName` from `loadClientRole()`.
Staff users (`user_type: portal_staff`) match role key `"staff"`.

Staff with `allowWrite: true` in `portalGrants` acts as a write-agent with no delegation
restrictions: `allowedSlugs` is ignored (`resolveRunDelegation` in `portal-agent-ctx.js`
returns `allowedSlugs: null`) — such a staff account is a workplace, not a delegating agent.

## Proxy Sub-Routers

### Teamchat Proxy (`/portal/api/teamchat`)

Built by `createPortalTeamchatRouter` (`teamchat/router.js`), mounted in `router.js` on `/api/teamchat`. The whole sub-router sits under `requirePortalJwt` — not `portalAuth('*')`, so `req.portalGrants` is **not** populated here; access is decided by portal config and room visibility, not by workspace grants. Rate limit 60 req/min.

- **Identity bridge:** `portal:<clientObjectId>` username mapping (`teamchat/identity.js`)
- **Access control:** room visible if listed in `portalConfig.teamchat.roomIds`, type=support, or visibility=public (`teamchat/access.js`)
- Proxies: rooms, topics, messages, reactions, stars, polls, receipts, reminders, file upload, agent listing

### Meta-KB Proxy (`/portal/api/meta-kb`)

Built by `createPortalMetaKbRouter` (`meta-kb-proxy.js`), mounted in `router.js` on `/api/meta-kb`. Rate limit 30 req/min.

Auth: the sub-router mounts `optionalPortalJwt`, then judges itself — `404 NOT_CONFIGURED` when `metaKb.enabled` is off, and `401` for an anonymous caller unless the method is read-only **and** `metaKb.anonymousRead === true`. Anonymous read is a separate flag on purpose: it is not implied by `agent.anonymousAllowed`.

- Endpoints: welcome, topics, debates, export (MD/DOCX), appropriation, analytics, iterations, changes
- `GET /workspace-tools` — list active workspace tools (for tool toggles UI)
- `PATCH /agents/:id/tools` — update internal agent's tools array (body: `{ tools: string[] }`)

### Decisions Proxy (`/portal/api/decisions`)

See detailed table below.

### Agent Registry Proxy (`/portal/api/agents`)

See detailed table below.

## Portal Config — proxy modules

Config keys controlling proxy sub-routers:

- `teamchat.enabled` — enable teamchat proxy
- `teamchat.roomIds` — array of room IDs visible to portal clients
- `metaKb.enabled` — enable meta-KB proxy
- `decisions.enabled` — enable decisions proxy
- `agent.enabled` — enable agent registry proxy and SSE agent/run endpoint

---

## Decisions Proxy (`/portal/api/decisions/*`)

**File:** `portal/decisions-proxy.js`
**Config flag:** `decisions.enabled`

| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | List decisions with filters (q, verdict, domain) |
| GET | `/graph` | Decision graph nodes+edges |
| GET | `/:id` | Single decision with links and discussions |
| GET | `/:id/links` | Decision links grouped by rel_type |
| GET | `/:id/discussions` | Linked teamchat discussions |
| GET | `/:id/history` | Change history |
| GET | `/:id/conflicts` | Conflict analysis |
| GET | `/:id/iterations` | Linked meta-kb iterations |
| GET | `/:id/kag-stats` | KAG entity/relation counts |

## Agent Registry Proxy (`/portal/api/agents/*`)

**File:** `portal/agent-registry-proxy.js`
**Config flag:** `agent.enabled`

| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | List all agents from `_v2_agent_registry` |
| GET | `/:id` | Get single agent with systemPrompt |
| PATCH | `/:id` | Update name, description, systemPrompt |

## Teamchat Proxy Additions

**File:** `portal/teamchat/router.js` — inside `createPortalTeamchatRouter`

| Method | Path | Description |
|--------|------|-------------|
| GET | `/rooms/:roomId/agents` | List agents in room enriched with registry metadata |

## Portal Config Additions

`KNOWN_PAGE_TYPES` in `config-utils.js` now includes: `teamchat`, `meta-kb`, `decisions`.

## Server Functions (codespace)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/portal/api/fn/:repo/:name` | `portalAuth('admin', 'owner', 'editor')` | Execute a codespace server function. Rate limit: 60/min. Body: JSON args. |
| POST | `/portal/api/agent/resume` | `requirePortalJwt` | Resume HITL approval. Body: `{ threadId, approved }`. |

Executes `api/<name>.js` from the specified codespace repo in an isolated V8 sandbox. See [`docs/guides/codespace-server-functions.md`](../../../docs/guides/codespace-server-functions.md) and [`codespace.md`](codespace.md) (Server Functions section).

## Portal config — write paths and merge semantics

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/portal/api/config` | workspace JWT + `requireRole('admin')` | Create/replace config (`upsertConfig` + `invalidatePortalCache`). Body: `{ config, active?, custom_domain? }`. Refs validated (`validatePortalRefs`) — 400 `INVALID_CONFIG_REFS`. |
| GET | `/portal/api/config/history` | workspace JWT + `requireRole('admin')` | Config snapshots, newest first (`?limit=`, 1–100, default 20). Secrets included — admin surface like `/api/config/full`. |
| POST | `/portal/api/config/restore` | workspace JWT + `requireRole('admin')` | Restore from snapshot. Body: `{ historyId }`. Goes through `upsertConfig` (the replaced config lands in history itself); ref validation deliberately skipped. |
| PUT | `/portal/api/config/active` | workspace JWT + `requireRole('admin')` | Flip `active` only. |

MCP tools `set_portal_config` / `update_portal_module` are wrappers over the same
`upsertConfig` + `invalidatePortalCache` path.

**Merge semantics (verified in `ai/agent/tools/portal.js`):**

- `set_portal_config(merge: true)` deep-merges objects, but **arrays are replaced
  wholesale** — a partial `pages[]` wipes every page. For a one-module change use
  `update_portal_module` instead.
- `update_portal_module(slug, config, moduleIndex?)` shallow-merges into
  `modules[moduleIndex ?? 0].config`: keys you pass are replaced, keys you omit are
  preserved. `moduleIndex` is positional; the first module is the default.
- Both MCP write tools and both REST routes run `upsertConfig` +
  `invalidatePortalCache` — the Valkey SSR cache (`portal_html:<db>:<path>`, 300 s)
  is dropped by the write itself; direct DB writes bypass this and read stale for
  up to 5 minutes.
- REST fallback when MCP tooling is unavailable: workspace admin JWT
  (`POST /api/v2/iam/login`) → `POST /api/v2/:db/portal/api/config`. Same
  validation (`preserveSecrets`, `validatePortalConfig`), same cache invalidation.

## PM API (workspace PM module over portal JWT)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/portal/api/pm/statuses` | `portalAuth('*')` (any authenticated) | Live PM status list (`pm/statuses.listStatuses`) — the source of TaskBoard columns (name, label, kind, order). `?board_id=` for one board's statuses, omitted — the whole workspace (same contract as `GET /api/v2/:db/pm/statuses`). Bad `board_id` — 400 `INVALID_BOARD_ID`. Rate limit: 120/min. |
| GET | `/portal/api/pm/issues` | `portalAuth('*')` (any authenticated) | List PM issues via `pm/service.listIssues` (offset+limit, honest `total`, cap 200). Filters pass through: `status`, `type`, `priority`, `assigneeId`, `sprintId`, `parentId`, `search`, `labels`, `limit`, `offset`. Each item carries `assignee_email` (route-level join on `_v2_users`) — the key portals use to map issues onto their member records. Rate limit: 120/min. |
| POST | `/portal/api/pm/issues` | `portalAuth('admin', 'owner', 'editor')` | Create issue via `pm/service.createIssue`, `reporter_id` = portal user id. Body whitelisted to: `title` (required, non-empty), `description`, `status`, `priority`, `assigneeId`, `dueDate`, `sortOrder`. Rate limit: 30/min. |
| PATCH | `/portal/api/pm/issues/:id` | `portalAuth('admin', 'owner', 'editor')` | Update issue via `pm/service.updateIssue` (same body whitelist; `title`, when present, must be non-empty). Rate limit: 60/min. |

Both write routes reject with `403 PM_NO_WORKSPACE_USER` when the portal JWT carries no workspace user id (portal-customer accounts): `reporter_id` is an FK to `_v2_users`. Lazy PM table init runs via `ensureSideTables(pool, db, 'pm')` on every route, same as the module's own router. PM statuses, roles and table DDL still live in [`pm.md`](pm.md) — this surface only exposes the existing service under the portal cookie; PM is the source of truth for portal task boards (portal-kit source `pm-issues:0`, widget `TaskBoard`).

## Custom Code Portal (codespace repo)

6 Vue SFC files loaded via vue3-sfc-loader with relative imports:
- `main.vue` — app shell and routing
- `MetaKbView.vue` — meta-kb debates and topics
- `TeamchatView.vue` — teamchat room view
- `DecisionsView.vue` — decisions browser
- `ChatView.vue` — SSE chat with agent
- `CitationRenderer.vue` — structured source parsing with [N] badges

**URL routing:** `?_cc_view=` and `?_cc_sub=` params via `props.api.navigate()`

**SSE chat:** `POST /portal/api/agent/run` with `agentSlug: 'meta-kb'`

**Agent settings:** edit name/description/systemPrompt via `/portal/api/agents`

**CitationRenderer:** structured source parsing with `[N]` badges for inline citations

### How section code updates reach clients

The compiled-SFC cache (`sfc:` in localStorage) is keyed by a **hash that includes
the file content** (vue3-sfc-loader `compiledCache` pattern: key = hash of loader
version + path + source + compile options), and section files are fetched with
`cache: 'no-cache'` (revalidate). Consequence: committing new `.vue` content to
the configured ref is enough — clients compile the new source on their next page
load; upgrading vue3-sfc-loader itself also invalidates the cache (loader version
is part of the key). Neither a `ref` bump nor an `SFC_CACHE_VERSION` bump is
required for content changes; bump `SFC_CACHE_VERSION` only when the
compile-time options change in `CustomCodeRuntime.vue`. Section styles go through
the same blob route (`Cache-Control: no-cache` + ETag = commit sha,
`portal/router.js:3069-3082`), so they revalidate the same way.
`assets/kit/<версия>/` is the exception: immutable address, content changes
require a new version.
