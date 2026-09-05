# Module: ws

**Path:** `src/api/v2/ws.js`
**URL:** `ws[s]://<host>/api/v2/:db/ws`
**Auth:** JWT required (first message after connect).

## Purpose

Real-time WebSocket server providing live data updates and collaborative editing. Clients subscribe to channels and receive push notifications when data changes. Also carries bidirectional ops for collaborative document editing, Yjs sandbox cells, teamchat rooms, WebRTC call signalling, and presence on objects and work points.

## Protocol

```
Client connects to /api/v2/:db/ws
Auth:        {"type":"auth","token":"<JWT>"}
Subscribe:   {"type":"subscribe","channel":"objects","typeId":123}
Unsubscribe: {"type":"unsubscribe","channel":"objects","typeId":123}
Server push: {"type":"change","channel":"objects","action":"create|update|delete","data":{...}}
```

### Message types (client → server)

Twelve, all dispatched from the single `switch (msg.type)` in the `message`
handler. Anything else gets `{"type":"error","code":"UNKNOWN_TYPE"}`.

| type | Handler | Description |
|------|---------|-------------|
| `auth` | `handleAuth` | Authenticate with JWT token |
| `subscribe` | `handleSubscribe` | Subscribe to a channel |
| `unsubscribe` | `handleUnsubscribe` | Unsubscribe from a channel |
| `documents` | `handleDocumentsOp` | Collaborative editing op (`delta-update`, `cursor-move`, `block-config-change`, `flush`, `presence-ping`) |
| `sandbox-collab` | `handleSandboxCollabOp` | Yjs CRDT sync for sandbox code cells (`yjs-update`, `yjs-sync`, `awareness-update`, `presence-ping`) |
| `objects` | `handleObjectsOp` | Objects-channel op (`cell-focus`, `cell-blur`, `presence-ping`) |
| `tc:join-room` | `handleTCJoinRoom` | Teamchat: join room `roomId`, subscribe to `tc:room:<roomId>` |
| `tc:leave-room` | `handleTCLeaveRoom` | Teamchat: leave room, replies `{"type":"tc:left"}` |
| `tc:typing` | `handleTCTyping` | Teamchat: relay typing indicator to the rest of the room |
| `calls` | `handleCallsMessage` | WebRTC signalling. Dropped silently when the workspace has `modules.calls === false` |
| `point` | `handlePointOp` | Work-point op (`intent`, `presence-ping`) — see [presence.md](presence.md) |
| `ping` | — | Keepalive; server replies `{"type":"pong"}` |

### Connection lifecycle

1. Client connects (TCP/TLS upgrade)
2. Auth timeout starts (10 seconds) — connection closed with code 4001 if no auth received
3. Client sends `{"type":"auth","token":"..."}` — server verifies JWT, loads user grants
4. Server replies `{"type":"auth","ok":true,"user":{...}}`
5. Client subscribes to channels
6. Server relays change events and ops

### Rate limiting

120 messages per 60-second window per connection. Excess messages close the connection with code 1008.

## Channels

`handleSubscribe` / `handleUnsubscribe` know four channel names: `objects`
(the default when `channel` is omitted), `documents`, `sandbox-collab`, `point`.
Any other value falls through to the generic branch and gets a subscription key
`<channel>:<typeId|*>:<parentId|*>` that nothing ever broadcasts to.

### objects channel

Subscription key format: `objects:<typeId|*>:<parentId|*>`

```json
// Subscribe
{"type":"subscribe","channel":"objects","typeId":42}

// Server broadcast on data change
{"type":"change","channel":"objects","action":"create","data":{"id":1,"typeId":42,...}}

// Objects ops
{"type":"objects","operation":"cell-focus","typeId":42,"rowId":10,"colId":5}
{"type":"objects","operation":"cell-blur","typeId":42,"rowId":10,"colId":5}
{"type":"objects","operation":"presence-ping","typeId":42}
```

### documents channel

Subscription key format: `documents:<workspaceId>:<documentId>`

```json
// Subscribe
{"type":"subscribe","channel":"documents","workspaceId":"myws","documentId":"123"}

// Collaborative ops
{"type":"documents","operation":"delta-update","workspaceId":"myws","documentId":"123","delta":{...}}
{"type":"documents","operation":"cursor-move","workspaceId":"myws","documentId":"123","position":{...}}
{"type":"documents","operation":"block-config-change","workspaceId":"myws","documentId":"123","blockId":"b1","config":{...}}
{"type":"documents","operation":"flush","workspaceId":"myws","documentId":"123"}
{"type":"documents","operation":"presence-ping","workspaceId":"myws","documentId":"123"}
```

Server relays ops to all other subscribers of the same document (sender excluded). `cursor-move` payloads include sender `userId` and `username` for collaborator cursor rendering.

### sandbox-collab channel

Subscription key format: `sandbox-collab:<topicId>:<messageId>`

```json
// Subscribe — both ids are mandatory, no wildcard form
{"type":"subscribe","channel":"sandbox-collab","topicId":"7","messageId":"42"}

// Ops
{"type":"sandbox-collab","operation":"yjs-update","topicId":"7","messageId":"42","data":"..."}
{"type":"sandbox-collab","operation":"yjs-sync","topicId":"7","messageId":"42","data":"..."}
{"type":"sandbox-collab","operation":"awareness-update","topicId":"7","messageId":"42","data":"..."}
{"type":"sandbox-collab","operation":"presence-ping","topicId":"7","messageId":"42"}
```

Ops are relayed to the other subscribers of the same cell with `userId`,
`username` and `ts` attached. `presence-ping` additionally answers the sender
with `{"type":"sandbox-collab","operation":"presence-list","users":[...]}`.

### point channel

"Who is on this work point right now." Full protocol, persistence and DB layout:
[presence.md](presence.md). Client: `frontend/src/composables/usePointPresence.js`.

Subscription key format: `point:<pointType>:<pointId>`

```json
// Subscribe — a plain `subscribe` frame, NOT a `point` frame
{"type":"subscribe","channel":"point","pointType":"record","pointId":123}

// Unsubscribe — the client must send this itself (see below)
{"type":"unsubscribe","channel":"point","pointType":"record","pointId":123}

// Client ops
{"type":"point","operation":"intent","pointType":"record","pointId":123,"what":"editing","ref":"col:5"}
{"type":"point","operation":"presence-ping","pointType":"record","pointId":123}
```

Server → client operations: `presence-join`, `presence-leave`, `presence-list`
(sent to the subscriber right after `subscribed`), `presence-ping` and `intent`
(relayed with `what` / `ref`).

**One validator, three entry points.** `_parsePointRef` is called from
`handleSubscribe`, `handleUnsubscribe` and `handlePointOp`. `pointType` must be
in `POINT_TYPES` from `modules/presence/service.js` — currently exactly `table`
and `record` — and `pointId` must be a positive integer. The check is not
defensive politeness: the values land in `_v2_presence_log` where `point_id` is
`BIGINT NOT NULL`, the INSERT is batched per workspace, and one `NaN` takes the
whole flush window down with it.

**The client sends its own unsubscribe.** `WsClient.unsubscribe()` in the
frontend only clears its local map — it writes nothing to the server. Until an
explicit `unsubscribe` frame arrives, peers get no `presence-leave` and the log
records no `leave`, all the way until the socket closes. `leave` is therefore
written in two places: the `unsubscribe` handler (only if the subscription was
actually held, so no double-leave) and the `close` handler, which walks the
remaining `point:*` subscriptions.

## Channel Authorization

Authorization is checked once at subscribe time (industry standard — mirrors Laravel Echo/Pusher pattern). Broadcast is not filtered.

### Objects channel

When `typeId` is specified, the server checks `ws._user.grants` (loaded at auth time from `_v2_grants`):

- Specific grant (`grants[typeId]`): must be `READ` or `WRITE`
- Wildcard grant (`grants[0]`): must be `READ` or `WRITE`
- No numeric grants at all: allowed (legacy workspace fallback)

Error response on deny: `{"type":"error","code":"FORBIDDEN","message":"No read access to this type"}`

Wildcard subscriptions (no `typeId`) are always allowed.

### Documents channel

When `documentId` is specified, `documents/service.js → checkAccess(pool, ws._db, documentId, userId, 'viewer')` is called (imported into `ws.js` as `checkDocAccess`) (note: `ws._db` — the resolved `db_name`, not the URL slug):

- Document owner → allowed
- `is_public` document → allowed (viewer role)
- Explicit sharing entry (role >= viewer) → allowed
- No access → `{"type":"error","code":"FORBIDDEN","message":"No access to this document"}`

Wildcard `documents:*:*` (sidebar metadata) is always allowed.

### point channel

`canSubscribePoint(ws, db, pointType, pointId)`. Judges are reused, not invented:

- `pointType === 'table'` → `checkGrant(pool, db, grants, pointId, pointId, 'READ', …)` —
  literally the REST judge, the one `listObjects` uses.
- `pointType === 'record'` → one query resolves the record's type
  (`SELECT t FROM <eav> WHERE id = ?`), then `checkGrant` with that type, then
  `checkRowPermission(..., 'READ', ...)` — the row-level rules, as in
  `getObject`. A record that does not exist is denied.

The gate leans on `checkGrant`, not on the objects-channel `canSubscribeObjects` with its
"no numeric grants at all → allow" fallback. That became possible only once the socket's
grant map started being built by the same `buildRoleGrants` REST uses — before that it came
from a second builder and was empty for almost everyone, so a `checkGrant` gate would have
denied everybody except the owner and admins. The two maps are held together by
`__tests__/ws-rest-grants-parity.test.js`. Consequence: WS is neither stricter nor laxer
than REST.

Denied: `{"type":"error","code":"FORBIDDEN","message":"No read access to this point"}`.
Any exception inside the gate is logged and **denies** the subscription.

Without this gate a member whom `_v2_row_rules` forbid to read a record still saw
the names of everyone viewing it in `presence-list`, appeared there himself, and
received the free-text `what` / `ref` relayed from other people's `intent` frames.

### sandbox-collab channel

Validates only that `topicId` and `messageId` are present — no per-cell
authorization. [ADR-010](../../../docs/adr/010-ws-subscribe-authorization.md)
requires every channel to carry a subscribe gate; check the live
`handleSubscribe` branch before relying on one.

### Ops sender check

Four handlers verify the sender holds the subscription before relaying:

| Handler | Behaviour when the sender is not subscribed |
|---------|---------------------------------------------|
| `handleDocumentsOp` | `{"code":"NOT_SUBSCRIBED","message":"Subscribe to this document first"}` |
| `handleSandboxCollabOp` | `{"code":"NOT_SUBSCRIBED","message":"Subscribe to this cell first"}` |
| `handleObjectsOp` | `{"code":"NOT_SUBSCRIBED","message":"Subscribe to this channel first"}` |
| `handlePointOp` | `{"code":"NOT_SUBSCRIBED","message":"Нет подписки на точку <pointType>:<pointId> — сперва subscribe."}` |

`point` used to be the outlier here, returning silently, and that cost debugging time: a
client whose subscribe frame was rejected, or which unsubscribed and kept sending `intent`,
got no feedback at all — its ops simply vanished, indistinguishable from delivery. It now
answers like the other three.

## Slug vs db_name

The URL slug (`test-auto-service`) may differ from the internal `db_name` (`test_auto_service`). `handleAuth` resolves the slug and updates `ws._db` to the canonical `db_name`. All handlers that access `dbClients` or call service functions **must use `ws._db`**, not the closure-captured `db` variable (which holds the original slug).

## Document Delta Persistence

WS is the primary save channel for collaborative documents (Hocuspocus/Figma-style):

1. Each incoming `delta-update` is composed into an in-memory buffer (synchronous — no race)
2. Delta is relayed to peers immediately (no DB wait)
3. Debounce: flush to DB 2s after last edit
4. Max debounce: force flush every 10s during continuous typing
5. On disconnect: immediate flush to avoid data loss on page reload
6. On server shutdown (SIGTERM/SIGINT): flush all pending buffers

## Exports

Fifteen. Five of them are dispatched internally from the message switch and are
exported **only so tests can drive them directly** — `export` cannot be dropped
from those without breaking the suites listed below.

| Export | Description | Used by |
|--------|-------------|---------|
| `initWebSocket()` | Initialize WSS (call once on startup) | `scripts/start.js`, `scripts/start-pg.js` |
| `handleV2Upgrade(req, socket, head)` | Handle HTTP upgrade for WS connections | same |
| `isV2WsPath(pathname)` | Test if a URL path matches the WS route | same |
| `broadcast(db, channel, action, data)` | Push a change event to subscribed clients | `api/v2/index.js`, `utils/ws-listeners.js`, `files/doc-processor.js`, `ai/autofill-worker.js` |
| `broadcastAll(channel, action, data)` | Push to all clients across all workspaces | `utils/ws-listeners.js` |
| `broadcastToRoom(db, subKey, message)` | Push a raw message to one subscription key | `teamchat/listeners.js` |
| `broadcastDocumentDelta(db, wsId, docId, delta)` | Inject AI-generated delta into a document room | *no caller in `src/` or `scripts/`* |
| `flushDocBeforeRead(db, documentId)` | Flush pending buffer before REST GET | `documents/router.js` |
| `invalidateDocBuffer(db, documentId)` | Invalidate in-memory base delta cache | `documents/router.js` (dynamic import) |
| `getClientCount()` | Return total connected client count | *no caller in `src/` or `scripts/`* |
| `canSubscribeObjects(grants, typeId)` | Check if grants allow subscribing to a typeId | tests only — `__tests__/ws-grants.test.js` |
| `handleAuth(ws, db, token, authTimeout)` | `auth` frame handler | tests only — `__tests__/ws-auth-access.test.js`, `ws-membership.test.js` |
| `handleSubscribe(ws, db, msg)` | `subscribe` frame handler | tests only — `presence/__tests__/ws-point-*.test.js` |
| `handleUnsubscribe(ws, msg)` | `unsubscribe` frame handler | tests only — same |
| `handlePointOp(ws, db, msg)` | `point` frame handler | tests only — `presence/__tests__/ws-point-validation.test.js` |
