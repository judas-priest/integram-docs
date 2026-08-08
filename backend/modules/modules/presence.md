# Module: presence

**Path:** `src/api/v2/modules/presence/`
**Files:** `router.js`, `service.js`
**Base URL:** `/api/v2/:db/presence`
**Auth:** JWT required. POST: editor+, GET: viewer+.

## Purpose

Tracks real-time user presence on EAV objects (e.g. who is viewing or editing a record). Events are logged to a per-workspace table and broadcast over WebSocket point channels.

## REST Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/:db/presence` | Log a presence event |
| GET | `/:db/presence` | Query presence feed |

### POST `/:db/presence`

Body:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `point_type` | string | yes | Entity type (e.g. `object`) |
| `point_id` | int | yes | Entity ID |
| `actor` | string | yes | Username |
| `kind` | enum | yes | `intent`, `act`, `join`, `leave`, `help` |
| `zone` | string | no | UI zone within the point |
| `what` | string | no | Action description |
| `ref` | string | no | Reference context |
| `peer` | string | no | Peer identifier |

Writes are batched: 2-second debounce, flush at 50 events.

### GET `/:db/presence`

Query params: `point_type`, `point_id`, `actor`, `limit`, `offset`.

Returns logged presence events from `_v2_presence_log`.

## WebSocket Point Channel

Handled in `ws.js` (lines 1390–1506). Clients subscribe to a point channel and receive real-time presence notifications.

### Subscribe

```json
{ "type": "point", "operation": "subscribe", "pointType": "object", "pointId": 123 }
```

### Operations

| Operation | Direction | Description |
|-----------|-----------|-------------|
| `presence-join` | server → client | User joined the point |
| `presence-leave` | server → client | User left the point |
| `presence-list` | server → client | Current presence list for the point |
| `presence-ping` | client → server | Heartbeat to maintain presence. Server re-broadcasts the ping to all other peers subscribed to the same point channel (excluding the sender), so they can update the user's last-seen timestamp. |
| `intent` | client → server | User declared an intent (e.g. editing a field) |

### Message Shape

```json
{
  "type": "point",
  "operation": "presence-join",
  "pointType": "object",
  "pointId": 123,
  "userId": 42,
  "username": "alice",
  "ts": 1718300000000
}
```

Presence events are persisted to `_v2_presence_log` via a fire-and-forget REST POST.

WS generates only `join`, `leave`, `intent`. REST API accepts the full set (`act`, `help` in addition) for use by frontends and automations (e.g. logging task completion or help requests).

## DB Table

`_v2_presence_log` (per-workspace, lazy-initialized):

| Column | Type | Description |
|--------|------|-------------|
| `id` | serial | Primary key |
| `point_type` | text | Entity type |
| `point_id` | int | Entity ID |
| `actor` | text | Username |
| `kind` | text | `intent`, `act`, `join`, `leave`, `help` |
| `zone` | text | UI zone (nullable) |
| `what` | text | Action description (nullable) |
| `ref` | text | Reference context (nullable) |
| `peer` | text | Peer identifier (nullable) |
| `created_at` | timestamptz | Event timestamp |
