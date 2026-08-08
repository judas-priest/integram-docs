# Module: ws

**Path:** `src/api/v2/ws.js`
**URL:** `ws[s]://<host>/api/v2/:db/ws`
**Auth:** JWT required (first message after connect).

## Purpose

Real-time WebSocket server providing live data updates and collaborative editing. Clients subscribe to channels and receive push notifications when data changes. Also supports bidirectional ops for collaborative document editing and object-level presence.

## Protocol

```
Client connects to /api/v2/:db/ws
Auth:        {"type":"auth","token":"<JWT>"}
Subscribe:   {"type":"subscribe","channel":"objects","typeId":123}
Unsubscribe: {"type":"unsubscribe","channel":"objects","typeId":123}
Server push: {"type":"change","channel":"objects","action":"create|update|delete","data":{...}}
```

### Message types (client → server)

| type | Description |
|------|-------------|
| `auth` | Authenticate with JWT token |
| `subscribe` | Subscribe to a channel |
| `unsubscribe` | Unsubscribe from a channel |
| `documents` | Send a collaborative editing op (delta-update, cursor-move, etc.) |
| `objects` | Send an objects-channel op (cell-focus, cell-blur, presence-ping) |
| `ping` | Keepalive — server replies with `{"type":"pong"}` |

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

When `documentId` is specified, `checkAccess(pool, ws._db, documentId, userId, 'viewer')` is called (note: `ws._db` — the resolved `db_name`, not the URL slug):

- Document owner → allowed
- `is_public` document → allowed (viewer role)
- Explicit sharing entry (role >= viewer) → allowed
- No access → `{"type":"error","code":"FORBIDDEN","message":"No access to this document"}`

Wildcard `documents:*:*` (sidebar metadata) is always allowed.

### Ops sender check

`handleObjectsOp` and `handleDocumentsOp` verify the sender is subscribed to the target channel before relaying. Unsubscribed senders receive `{"type":"error","code":"NOT_SUBSCRIBED","message":"Subscribe to this channel first"}`.

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

### Exported helpers

| Export | Description |
|--------|-------------|
| `initWebSocket()` | Initialize WSS (call once on startup) |
| `handleV2Upgrade(req, socket, head)` | Handle HTTP upgrade for WS connections |
| `isV2WsPath(pathname)` | Test if a URL path matches the WS route |
| `broadcast(db, channel, action, data)` | Push a change event to subscribed clients |
| `broadcastAll(channel, action, data)` | Push to all clients across all workspaces |
| `broadcastDocumentDelta(db, wsId, docId, delta)` | Inject AI-generated delta into document room |
| `flushDocBeforeRead(db, documentId)` | Flush pending buffer before REST GET |
| `invalidateDocBuffer(db, documentId)` | Invalidate in-memory base delta cache |
| `getClientCount()` | Return total connected client count |
| `canSubscribeObjects(grants, typeId)` | Check if grants allow subscribing to a typeId |
