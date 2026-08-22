# Module: admin

**Path:** `src/api/v2/modules/admin/`
**Files:** `router.js`, `service.js`, `qa-bugs.js`
**Base URL:** `GET|POST|DELETE /api/v2/:db/admin/...`
**Auth:** All endpoints require JWT. Most require `admin` role.

## Purpose

Workspace administration: backup/restore, grants, roles, row-level permission rules, dead letter queue management, BKI integration, permissions report, and QA tooling.

## Endpoints

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/admin/grants` | any | Get current user's grant list |
| POST | `/admin/grants/check` | any | Check if user has a specific grant on an object |
| POST | `/admin/backup` | admin | Create ZIP backup of workspace data |
| POST | `/admin/restore` | admin | Restore workspace from ZIP backup (file upload or base64 content) |
| GET | `/admin/export/:typeId` | admin | Export all objects of a type (JSON) |
| GET | `/admin/export-all` | admin | Export all types metadata |
| POST | `/admin/databases` | admin | **Deprecated** — create workspace (use `/workspaces` instead) |
| GET | `/admin/bki-export` | admin | BKI (credit bureau) structured export (ZIP) |
| POST | `/admin/bki-import` | admin | BKI structured import (file upload) |
| GET | `/admin/row-rules` | admin | List row-level permission rules (`?typeId=N`) |
| POST | `/admin/row-rules` | admin | Create a row-level rule |
| DELETE | `/admin/row-rules/:id` | admin | Delete a row-level rule |
| GET | `/admin/permissions-report` | admin | Full matrix: users → roles → grants. `?format=csv` for CSV |
| GET | `/admin/roles` | admin | List roles for this workspace |
| POST | `/admin/roles` | admin | Create custom role |
| PUT | `/admin/roles/:id` | admin | Update custom role name/description |
| DELETE | `/admin/roles/:id` | admin | Delete custom role (system roles cannot be deleted) |
| GET | `/admin/v2-grants` | admin | List grants from `_v2_grants` table |
| POST | `/admin/v2-grants` | admin | Create or update a grant |
| DELETE | `/admin/v2-grants/:id` | admin | Delete a grant |
| GET | `/admin/dlq` | admin | List failed BullMQ jobs |
| POST | `/admin/dlq/:jobId/retry` | admin | Retry a DLQ job |
| POST | `/admin/behavioral/trigger` | admin | Dev: manually trigger behavioral pattern extraction |
| GET | `/admin/qa-bugs` | admin | List QA bug reports (only when `QA_BUGS_ENABLED=true`) |
| GET | `/admin/qa-results` | admin | All QA test sessions + results from all users |

## Key Concepts

**Backup/restore** — serializes all EAV objects and requisites for a workspace into a ZIP file. Restore supports file upload, a path to a backup file on disk, or raw content in request body. Path is validated to prevent directory traversal. Uses `getPoolForDb()` for workspace-specific connections, supporting workspaces with remote databases.

**Row-level rules** — stored in `_v2_row_rules`. Types: `OWNER_ONLY` (only creator can read/write), `FILTER` (SQL-level filter by user attribute), `DENY_ALL`. Applied at query time in `utils/row-permissions.js`.

**Grants** (`_v2_grants`) — role-based ACL per table (`targetTypeId=0` means all tables). Fields: `level` (READ/WRITE), `canExport`, `canDelete`, `maskFilter` (column masking). Emits `grants.changed` event on change to clear role cache.

**Roles** (`_v2_roles`) — workspace-scoped. System roles (`is_system=true`) cannot be modified or deleted. The `public` system role is auto-created for each workspace and defines grants for anonymous portal access.

**Portal grant resolution** — `resolvePortalUserGrants(pool, workspace, portalClient)` in `utils/v2-grants.js` resolves `_v2_grants` for portal users: staff users resolve via `_v2_memberships` role → `_v2_grants`; customers and anonymous users resolve via the `public` role's grants. Returns `{ role, grants }` in legacy format.

**BKI** — Russian credit bureau integration. Export produces ZIP with XML in required format; import accepts ZIP or XML.

## Dependencies

- `utils/row-permissions.js` — row rule CRUD
- `utils/v2-grants.js` — grant CRUD
- `utils/queue.js` — DLQ listing and retry
- `modules/swarm-memory/behavioral-collector.js` — dev trigger
- Event bus: emits `grants.changed`

## DB Tables

- `_v2_roles` (global/public schema)
- `_v2_grants` (global/public schema)
- `_v2_row_rules` (per-workspace, lazy-init)
- `qa_test_sessions`, `qa_test_results` (global, only when QA enabled)
