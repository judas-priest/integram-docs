# Module: orgs

**Path:** `src/api/v2/modules/orgs/`
**Files:** `router.js`, `service.js`
**Base URL:** `/api/v2/orgs/...`  (no `:db` — global)
**Auth:** JWT required for all endpoints.

## Purpose

Organizations group workspaces under a shared identity (company or team). An org has a unique slug and a member list with roles. Workspace creation can reference an `org_id`.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/orgs` | List orgs current user belongs to |
| POST | `/orgs` | Create org (`name`, `slug`) |
| GET | `/orgs/:slug` | Get org details |
| PUT | `/orgs/:slug` | Update org name |
| DELETE | `/orgs/:slug` | Delete org (owner only) |
| GET | `/orgs/:slug/members` | List org members |
| POST | `/orgs/:slug/members` | Add member by email (`role`: owner/admin/editor/viewer) |
| PUT | `/orgs/:slug/members/:id` | Update member role |
| DELETE | `/orgs/:slug/members/:id` | Remove member |

## Slug Rules

- 3–64 characters
- Lowercase alphanumeric, `-`, `_`
- Must start with a letter
- Regex: `/^[a-z][a-z0-9_-]{1,62}[a-z0-9]$/`

## Roles

`owner` | `admin` | `editor` | `viewer` — same role hierarchy as workspaces.

Only the org owner can delete the org. Members can only be managed by org owner/admin.

## DB Tables

- `_v2_orgs` (global public schema) — `id`, `name`, `slug`, `owner_id`, `created_at`
- `_v2_org_members` (global public schema) — `org_id`, `user_id`, `role`, `joined_at`
