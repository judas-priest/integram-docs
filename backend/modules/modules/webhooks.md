# Module: webhooks

**Path:** `src/api/v2/modules/webhooks/`
**Files:** `router.js`, `service.js`, `worker.js`, `listeners.js`
**Base URL:** `/api/v2/:db/webhooks/...`
**Auth:** JWT required. Create/update/delete: `admin`. List webhooks: any. List deliveries: `admin`. Retry delivery: `admin`.

## Purpose

Outgoing HTTP notifications on EAV object CRUD events. When a subscribed event occurs, the backend enqueues a delivery job that POSTs a JSON payload to the configured URL. Supports HMAC signatures, retry with exponential backoff, and a delivery history log.

## Endpoints

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/webhooks` | any | List webhooks |
| POST | `/webhooks` | admin | Create webhook (idempotency key supported) |
| PATCH | `/webhooks/:id` | admin | Update URL, events, active flag, secret |
| DELETE | `/webhooks/:id` | admin | Delete webhook |
| GET | `/webhooks/:id/deliveries` | admin | Delivery history (default 50, max 200) |
| POST | `/webhooks/deliveries/:deliveryId/retry` | admin | Manually retry a failed delivery |

## Webhook Config

```json
{
  "url": "https://my-system.example.com/hook",
  "typeId": 123,
  "events": ["create", "update", "delete"],
  "secret": "my-hmac-secret"
}
```

- `typeId`: EAV type to watch. `0` = all types.
- `events`: any combination of `create`, `update`, `delete`
- `secret`: if set, `X-Integram-Signature: sha256=HMAC(secret, body)` is added to each delivery

## Delivery Payload

```json
{
  "event": "create",
  "typeId": 123,
  "objectId": 456,
  "db": "my-workspace",
  "timestamp": "2025-01-01T12:00:00Z",
  "data": { "Name": "Acme", "Status": "New" }
}
```

## Worker (`worker.js`)

BullMQ consumer. Each delivery attempt:
1. SSRF-checks the target URL via `utils/url-guard.js`
2. POSTs the JSON payload with HMAC signature header
3. On HTTP 2xx: marks delivery as `delivered`
4. On failure: retries up to 7 times with exponential backoff (2s base, jitter 0.5)
5. Failed deliveries go to DLQ

## Listeners (`listeners.js`)

Subscribes to `object.created/updated/deleted` events. For each matching webhook, enqueues a delivery job via BullMQ.

## Idempotency

`X-Idempotency-Key` supported on POST, cached 30 seconds.

## DB Tables

- `_v2_webhooks` (per-workspace) — webhook definitions with URL, events, secret
- `_v2_webhook_deliveries` (per-workspace) — delivery log: status, response code, duration, error
