# Module: notifications

**Path:** `src/api/v2/modules/notifications/`
**Files:** `router.js`, `service.js`
**Base URL:** `/api/v2/:db/notifications/...`
**Auth:** JWT required for all endpoints.

## Purpose

In-app notification feed for workspace users. Notifications are created by the system (automations, connectors, AI jobs, comments) and surfaced to the user. Supports approval actions for `wait_approval` automation steps.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/notifications` | List notifications. `?unreadOnly=true`, `?limit=`, `?offset=` |
| GET | `/notifications/count` | Unread count |
| POST | `/notifications/read-all` | Mark all as read |
| POST | `/notifications/:id/read` | Mark one notification as read |
| DELETE | `/notifications/:id` | Delete a notification |
| POST | `/notifications/:id/action` | Approve/reject a pending approval request |

## Notification Types

- `system` — connector runs, AI jobs, import results
- `webhook` — incoming webhook events
- `mention` — @mention in a comment or document
- `automation` — automation triggered a notification action
- `approval` — pending `wait_approval` automation step
- `insight` — AI-generated insights and recommendations

## Approval Flow

When an automation has a `wait_approval` action step, it creates a notification of type `approval` and suspends execution. When the user calls `POST /notifications/:id/action` with `{ action: "approve" | "reject" }`:
1. Handler calls `handleNotificationAction()` which returns `{ suspendedId, actionKey }`
2. Router emits `approval.resolved` on event bus
3. Automations listener resumes the suspended execution

## WebSocket

New notifications are delivered via two mechanisms:
- **`bus.emit('notification.created')`** — in-process delivery (HTTP server → `ws-listeners.js` → all open WS connections for the workspace user).
- **`pg_notify('v2_notification')`** — cross-process delivery (worker process → PostgreSQL LISTEN/NOTIFY → HTTP server picks up and forwards to WS). Used when notifications are created inside BullMQ workers (e.g. automation runs, connector jobs) that share no memory with the HTTP process.

## Mention Processing (`processMentions`)

`processMentions(db, pool, body, authorId)` parses `@[userId]` tokens from comment or document body text and creates a `mention` type notification for each mentioned user. Called automatically when a PR comment or object comment is saved. The format `@[42]` references user ID 42 — any user in the workspace can be mentioned. Duplicate mentions in the same body are deduplicated before notification creation.

## DB Tables

- `_v2_notifications` (per-workspace) — `id`, `user_id`, `username`, `type`, `title`, `body` (TEXT), `read_at` (TIMESTAMPTZ), `actions_cfg` (JSONB), `action_taken` (VARCHAR), `suspended_id` (BIGINT), `obj_id` (BIGINT), `ref_id` (BIGINT), `created_at`

## AI Tools

| Tool | Tier | Description |
|------|------|-------------|
| `list_notifications` | TIER_LOW | List notifications for the current user. Supports filtering by type and read status. |
| `mark_read` | TIER_MEDIUM | Mark one or all notifications as read. |
| `send_notification` | TIER_MEDIUM | Create and send a notification to a workspace user. |
| `delete_notification` | TIER_MEDIUM | Delete a notification by ID. |
| `notification_action` | TIER_HIGH | Execute an action on an approval notification (approve/reject). Emits `approval.resolved` event to resume suspended automation (both REST and MCP paths). |
