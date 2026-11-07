# Module: comments

**Path:** `src/api/v2/modules/comments/`
**Files:** `router.js`, `service.js`, `listeners.js`
**Base URL:** `/api/v2/:db/objects/:objId/comments/...`
**Auth:** JWT required for all endpoints.

## Purpose

Threaded comments on EAV objects. Supports nested replies (one level via `parentCommentId`) and emoji reactions.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/objects/:objId/comments` | List comments for an object |
| POST | `/objects/:objId/comments` | Create a comment (`body`, optional `parentCommentId`) |
| PATCH | `/objects/:objId/comments/:commentId` | Edit comment body (own comments only) |
| DELETE | `/objects/:objId/comments/:commentId` | Delete comment (own comments only — admins have no override) |
| GET | `/objects/:objId/comments/:commentId/reactions` | List reactions |
| POST | `/objects/:objId/comments/:commentId/reactions` | Add emoji reaction |
| DELETE | `/objects/:objId/comments/:commentId/reactions/:emoji` | Remove emoji reaction |

## Data Model

```
Comment {
  id, obj_id, parent_comment_id, body (text, max 10000),
  author (username), created_at, updated_at
}

Reaction {
  comment_id, author (VARCHAR(128), username), emoji (VARCHAR(16)), created_at
  UNIQUE (comment_id, author, emoji)
}
```

## Listeners (`listeners.js`)

Subscribes to `comment.created` event to:
- Trigger `@mention` notifications
- Create backlinks via `utils/record-mentions.js`

## Event Emitted

On `createComment`: emits `comment.created` with `{ db, pool, objId, commentId, body, authorId }`.

## DB Tables

- `_v2_comments` (per-workspace) — comment threads
- `_v2_reactions` (per-workspace) — emoji reactions per comment per user
