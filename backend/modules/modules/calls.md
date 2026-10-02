# calls module

Pure P2P 1-1 calls and a single per-workspace voice room (max 12).
Per-workspace toggle via `workspace.settings.modules.calls`.

## Files

- `router.js` — REST: `GET /:db/calls/turn-credentials`, `GET /:db/calls/signal-token`, `GET /:db/calls/history`
- `service.js` — history + participants queries, status transitions
- `signalling.js` — WS handlers for channel `calls` (direct + room)
- `turn-credentials.js` — HMAC-SHA1 ephemeral creds
- `call-timeout.js` — `CallTimeoutManager`: 30-second ring timeout for pending calls
- `migrations.js` — idempotent satellite tables creation

## REST endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/:db/calls/turn-credentials` | Short-lived TURN creds for the current user |
| GET | `/:db/calls/signal-token` | Workspace-scoped JWT (1h TTL) for Go signal server auth. Contains `{sub, email, db}`. Signal server validates `db` claim matches requested workspace |
| GET | `/:db/calls/history?limit=50` | Call history rows where current user is initiator or target |

## WS channel `calls`

Plugged into existing `backend/src/api/v2/ws.js` via `case 'calls':`.

### Direct (1-1)
- `call.invite { targetUserId }` → server creates `calls_history` (status=pending), forwards to target
- `call.invite.ack` — server → initiator, confirms invite was sent `{ callId, targetUserId }`
- `call.accept|reject { callId }` — status transitions, forwarded to other party
- `call.cancel { callId }` — initiator → server, cancel a pending call (only initiator allowed)
- `call.timeout` — server → both parties, ring timeout expired (30s)
- `peer.offer|answer|ice|bye { callId, sdp|candidate }` — relayed to the other peer in the call
- On `peer.bye` — status=completed

### Workspace room
- `room.join { withVideo, withAudio }` — enforces ≤12, broadcasts presence, sends `peer.connect` to newcomer for each existing member
- `room.leave` / WS close — removes from presence; last-out closes the row (status=completed)
- `room.full { limit:12 }` — sent to client when limit reached
- `room.presence { users:[] }` — broadcast to all workspace clients
- `peer.connect { peerUserId, polite }` — server tells client "open a PC to this user". `polite = myUserId < peerUserId`
- `peer.offer|answer|ice|bye { peerUserId, ... }` — relayed to specific peer

## Call Timeout

`CallTimeoutManager` (in `call-timeout.js`) enforces a 30-second ring timeout for pending calls:
- On timeout: call status transitions to `missed`, both parties receive `call.timeout` WS action
- Creates a notification for the target user: "Пропущенный звонок"

## Disconnect Cleanup

On WebSocket disconnect, pending calls for the disconnected user are cleaned up based on role:
- **Initiator disconnects** → status becomes `cancelled`, target receives `call.cancel`
- **Target disconnects** → status becomes `missed`, initiator receives `call.timeout` and a "Пропущенный звонок" notification

Room memberships for the disconnected user are cleaned up (removed from presence, `room.leave` broadcast).

## Call Status Enum

`pending` → `active` | `rejected` | `cancelled` | `missed` | `failed`
`active` → `completed`

All statuses: `pending`, `active`, `completed`, `rejected`, `cancelled`, `missed`, `failed`.

## Data model

Satellite tables, per workspace:
- `calls_history(id, kind, initiator_user_id, target_user_id, started_at, ended_at, status, duration_sec)`
- `calls_participants(id, call_id, user_id, joined_at, left_at)`

## Env vars

- `TURN_SHARED_SECRET` (required for TURN endpoint)
- `TURN_URLS` (comma-separated)
- `TURN_TTL_SEC` (default 3600)

See `docs/ops/coturn-setup.md` for coturn config.
