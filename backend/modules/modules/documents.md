# Module: documents

**Path:** `src/api/v2/modules/documents/`
**Files:** `router.js`, `service.js`, `pdf.js`, `carbone.js`, `template-engine.js`, `doc-import.js`, `listeners.js`, `export.js` (экспорт документа в DOCX/PDF), `delta-to-markdown.js` (Quill Delta → markdown), `markdown-to-delta.js` (markdown → Quill Delta), `render.js` (рендер документа)
**Base URL:** `/api/v2/:db/documents/...`
**Auth:** JWT required. Write/delete: `editor`.

## Purpose

Block-based document editor (similar to Notion). Documents have a hierarchical block structure (Quill Delta). Supports real-time collaborative editing via WebSocket, versioning, folder organization, tags, sharing, and export to PDF/DOCX.

## Endpoints

### Documents

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/documents` | any | List documents (`?folderId=`, `?search=`). Workspace admin/owner sees ALL documents regardless of ownership, sharing, or public flag; other roles see only own + shared + public. |
| POST | `/documents` | editor | Create document |
| POST | `/documents/from-template/:templateId` | editor | Create document from workspace document template |
| POST | `/documents/from-system-template/:tplId` | editor | Create document from system (workspace) template |
| POST | `/documents/import` | editor | Import document from file (multipart) |
| GET | `/documents/:docId` | any | Get document metadata (flushes pending WS deltas first) |
| PUT | `/documents/:docId` | editor | Update title, folder, tags, icon, cover, is_template, is_public |
| DELETE | `/documents/:docId` | editor | Soft-delete document |
| GET | `/documents/trash` | viewer | List soft-deleted documents |
| POST | `/documents/:docId/restore` | editor | Restore soft-deleted document. Only the document owner can restore (non-owner editors are rejected). |
| DELETE | `/documents/:docId/purge` | editor | Permanently delete document |
| GET | `/documents/:docId/export/:format` | any | Экспорт Delta → markdown → DOCX/PDF (`format`: docx\|pdf); перед чтением смывает WS-буфер правок |

### Blocks

| Method | Path | Role | Description |
|--------|------|------|-------------|
| POST | `/documents/:docId/blocks` | editor | Append a block |
| PUT | `/documents/:docId/blocks/:blockId` | editor | Update block content/type/config |
| DELETE | `/documents/:docId/blocks/:blockId` | editor | Delete block |
| POST | `/documents/:docId/blocks/reorder` | editor | Reorder blocks |
| GET | `/documents/:docId/blocks/:blockId/history` | any | Block edit history |
| PUT | `/documents/:docId/blocks/:blockId/restore/:versionId` | editor | Restore block to a previous version |

### Delta sync

| Method | Path | Description |
|--------|------|-------------|
| POST | `/documents/:docId/sync` | Sync HTML content to blocks |
| POST | `/documents/:docId/sync-delta` | Sync Quill Delta to document |
| GET | `/documents/:docId/delta` | Get current Quill Delta (flushes WS buffer first) |

### Versions

| Method | Path | Description |
|--------|------|-------------|
| GET | `/documents/:docId/versions` | List version history |
| GET | `/documents/:docId/versions/:versionId` | Get full version snapshot |
| DELETE | `/documents/:docId/versions` | Delete multiple versions (body: `{ ids }`) |
| POST | `/documents/:docId/restore/:versionId` | Restore document to a version |

### Page Settings

| Method | Path | Description |
|--------|------|-------------|
| GET | `/documents/:docId/settings` | Get document settings |
| POST | `/documents/:docId/settings` | Save document settings |

### Folders

| Method | Path | Description |
|--------|------|-------------|
| GET | `/documents/folders` | List folders |
| POST | `/documents/folders` | Create folder |
| PUT | `/documents/folders/:folderId` | Update folder name/parent |
| DELETE | `/documents/folders/:folderId` | Delete folder |

### Tags

| Method | Path | Description |
|--------|------|-------------|
| GET | `/documents/tags` | List tags |
| POST | `/documents/tags` | Create tag |
| DELETE | `/documents/tags/:tagId` | Delete tag |
| POST | `/documents/:docId/tags/:tagId` | Add tag to document |
| DELETE | `/documents/:docId/tags/:tagId` | Remove tag |

### Sharing

| Method | Path | Description |
|--------|------|-------------|
| GET | `/documents/:docId/sharing` | Get share access list |
| POST | `/documents/:docId/sharing` | Grant access to user (body: `{ user_id\|email, role }`) |
| PUT | `/documents/:docId/sharing/:sharingId` | Update access role |
| DELETE | `/documents/:docId/sharing/:sharingId` | Revoke access |

### Invite links

| Method | Path | Description |
|--------|------|-------------|
| GET | `/documents/:docId/invites` | List invite links for document |
| POST | `/documents/:docId/invites` | Create invite link |
| DELETE | `/documents/:docId/invites/:inviteId` | Revoke invite link |
| POST | `/documents/invites/:token/accept` | Accept invite (requires auth) |

### Template utilities

| Method | Path | Description |
|--------|------|-------------|
| GET | `/documents/variables/:typeId` | List template variables for an EAV type |
| POST | `/documents/:docId/preview` | Render template to HTML preview (body: `{ type_id, object_id }`) |

## Block Types

`text`, `heading` (h1–h3), `code`, `quote`, `list` (bullet/numbered), `todo` (checkbox), `image`, `embed`, `divider`

Block content stored as Quill Delta JSON.

## Collaborative Editing (WebSocket)

Document edits are synced in real-time via the WebSocket protocol. The server keeps an in-memory buffer of pending Quill Delta ops per document (`ws.js`). Before any REST read (`GET /:docId`, `GET /:docId/delta`) the buffer is flushed to DB via `flushDocBeforeRead`. After `POST /:docId/sync-delta` the buffer is invalidated so the next read re-fetches from DB.

## Versioning

Versions are created on delta-sync saves. Manual deletion is supported via `DELETE /documents/:docId/versions` with a list of IDs. Restore replaces current blocks with snapshot content via `POST /documents/:docId/restore/:versionId`.

## DB Tables (per-workspace, lazy-init)

- `_v2_documents` — `id`, `title`, `folder_id`, `parent_id`, `status`, `owner`, `is_template`, `template_id`, `bound_type_id`, `icon`, `cover_url`, `settings`, `sort_order`, `is_public`, `deleted_at`, `deleted_by`, `created_at`, `updated_at`
- `_v2_document_blocks` — `id`, `doc_id`, `type`, `content`, `content_format`, `block_order`, `meta`, `created_at`, `updated_at`
- `_v2_document_folders` — `id`, `name`, `parent_id`
- `_v2_document_tags` — `id`, `name`, `color`
- `_v2_document_tag_map` — `doc_id`, `tag_id`
- `_v2_document_versions` — `id`, `doc_id`, `title`, `snapshot`, `saved_by`, `created_at`
- `_v2_document_sharing` — `doc_id`, `user_id`, `role`
- `_v2_document_invite_links` — `doc_id`, `token`, `role`, `expires_at`, `created_by`
- `_v2_document_block_versions` — `block_id`, `doc_id`, `content`, `meta`, `saved_by`, `created_at`
- `_v2_document_ops` — pending collaborative editing operations buffer (used by the WebSocket sync layer)
