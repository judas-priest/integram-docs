# Module: admin

**Path:** `src/api/v2/modules/admin/`
**Files:** `router.js`, `service.js`, `backup-carry.js`, `backup-paths.js`, `backup-crypto.js`, `zip.js`, `qa-bugs.js`
**Base URL:** `GET|POST|DELETE /api/v2/:db/admin/...`
**Auth:** All endpoints require JWT. Most require `admin` role.

## Purpose

Workspace administration: backup/restore, grants, roles, row-level permission rules, dead letter queue management, BKI integration, permissions report, and QA tooling.

## Endpoints

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/admin/grants` | any | Get current user's grant list |
| POST | `/admin/grants/check` | any | Check if user has a specific grant on an object |
| POST | `/admin/backup` | admin | Full workspace backup as ZIP. `?includeFiles=true` also carries on-disk directories; body `{passphrase}` encrypts secret columns (without it they are **not** exported). What the copy left behind is reported in `X-Backup-*` response headers |
| POST | `/admin/restore` | admin | Restore workspace from a backup. **Dry run by default** — pass `?dryRun=false` to execute. Body: `mode` (`empty`\|`replace`), `confirm` (the word `ЗАМЕНИТЬ`, required for `replace`), `passphrase` |
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

**Backup/restore** — a full copy of the workspace, not just its EAV table: the data table, every base table of its schema, and its rows in shared (`public`) tables. The table list is never written in code — it comes from the same two declarations the workspace **purge** uses (`registry/carry-catalog.js` over `registry/workspace-ownership.js`), because a backup answers "what must survive losing the workspace" and purge answers "what does the workspace have". `registry/__tests__/backup-covers-purge.test.js` guards that equality. Rationale: `docs/adr/028-workspace-backup-is-not-carry.md`.

Taken in one `REPEATABLE READ READ ONLY` transaction. Values are exported as the database's own text (`::text`), never through driver deserialization. Secret columns do not leave without a passphrase and are named in the manifest and in the response. Archives live outside any directory that purge deletes (`backup-paths.js`).

The data table travels in the legacy delta form, which encodes an *increment* from zero with a step of at least one — a row with `id < 1` is not expressible in it. Such rows exist in legacy workspaces (three on the dev stand as of 2026-08-23, one row each). The backup **refuses** with `BACKUP_ID_UNREPRESENTABLE` naming the count and the lowest id, rather than dropping them silently: the restore could never see the shortfall, since the rows are absent from the file and the entry checksum still matches the manifest. Renumber the rows (children name a row's id in `up`) and take the copy again.

Restore only targets **its own** workspace — a copy carries installation-unique values (portal domain, webhook secret, form token). Copying to another workspace is `clone_workspace`. There is no merge: the target must be empty, or be cleared under `mode: 'replace'` plus the confirmation word. Clearing and inserting share one transaction. A dry run executes the whole restore and ends in `ROLLBACK`, so its numbers are real.

A secret the database *requires* (NOT NULL with no default — form token, invitation and share tokens, bot token and webhook secret, login token hashes) cannot come back from a passphrase-less copy: it is not in the file at all. Restore **re-issues** it and names it in `secretsRenewed`; links that carried the old value die. The alternative — refusing the restore — would make a passphrase-less copy unrestorable into any workspace that has a form, an invitation, a share link or a bot. Same answer as the industry gives: GitLab restores without `gitlab-secrets.json` and then resets the affected entries. A non-secret column that is missing from the copy gets `BACKUP_COLUMN_MISSING` naming schema, table and column — never a bare `23502`.

An archive without `manifest.json` is read as the legacy single-file form — but only if it looks like one. If it carries `ws/` or `out/` entries (which only a full backup produces) while the manifest is gone, restore refuses with `BACKUP_MANIFEST_MISSING` instead of falling back: the legacy reader would take `eav.dmp` and leave the other entries unparsed and unnamed anywhere. Measured on a lost probe workspace: 92 such entries, restore answered "1 table of 93, 8 rows, 0 unknown entries, 0 shortfall" — success — while views, roles, grants and memberships never came back.

Restore supports file upload, a path to a backup file on disk, or raw content in request body. Path is validated to prevent directory traversal. Uses `getPoolForDb()` for workspace-specific connections, supporting workspaces with remote databases.

Round-trip verification against a live database: `scripts/backup-roundtrip-check.mjs`.

**Row-level rules** — stored in `_v2_row_rules`. Types: `OWNER_ONLY` (only creator can read/write), `ROLE_ONLY` (only users whose role matches the rule's `role_id`), `FILTER` (SQL-level filter by user attribute), `DENY_ALL`. The `rule` field is accepted as a free-form string by the route — anything outside this set is rejected by the table's CHECK constraint. Applied at query time in `utils/row-permissions.js`.

**Grants** (`_v2_grants`) — role-based ACL per table (`targetTypeId=0` means all tables). Fields: `level` (NONE/READ/WRITE/ADMIN — NONE is a deny row that overrides the role grant for that type), `canExport`, `canDelete`, `maskFilter` (column masking). Rows may be role-scoped (`username IS NULL`, resolved from the membership role) or per-user (`username` set); the single map builder (`buildRoleGrants` in `utils/v2-grants.js`) merges per-user rows over role rows, so a per-user NONE wins on the same typeId. Cache invalidation differs by path: the REST admin endpoints emit the `grants.changed` event to clear the role cache, while the AI tools `set_grant`/`remove_grant` do not emit it — both call `invalidateGrantCache` directly. Since issue #696 invalidation is username-aware: per-user cache entries are swept on grant writes; the staff identity is resolved from the JWT username, falling back to email on the portal staff path.

**Roles** (`_v2_roles`) — workspace-scoped. System roles (`is_system=true`) cannot be modified or deleted. The `public` system role is auto-created for each workspace and defines grants for anonymous portal access.

Имя кастомной роли — отображаемая строка: буквы любого алфавита, цифры,
пробел, дефис, подчёркивание (до 64 символов). Ограничение: кастомная роль
сегодня НЕ даёт системного уровня (ROLE_HIERARCHY знает только
owner/admin/editor/viewer) — участник с кастомной ролью в членстве проходит
маршруты как безролосый. Рабочий паттерн: членство на системной роли
(например editor), кастомная роль — для матчинга `roles` в спеках процессов.
Полноценные уровни кастомных ролей — отдельное решение.

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

## Доменные (квартетные) роли — сужающий слой (issue #273)

Пользователь воркспейса может получить вторую роль поверх системной
(owner/admin/editor/viewer). Она хранится в `_v2_memberships.domain_role_id`
(кастомная роль `_v2_roles`, `is_system=false`) и ТОЛЬКО сужает гранты:
итоговый уровень на тип = min(membership, доменная) там, где доменная роль
имеет строку (конкретную или wildcard `target_type_id=0`); где строк нет —
наследуется membership. Owner/admin не сужаются. EXPORT/DELETE сужаются по AND:
непустая карта флагов доменной роли — исчерпывающий перечень, где флаг выживает.
Маски (`mask_filter`) доменных ролей каркасом не поддерживаются.

Эндпоинты:
- `GET /api/v2/:db/admin/domain-roles` — список назначений (admin);
- `PUT /api/v2/:db/admin/domain-role` `{username, roleId|null, workspaceId?}` —
  назначить/снять (admin, чужая область — только суперадмин).

AI-инструменты (dual path): `list_domain_roles` (TIER_LOW) и `set_domain_role`
(TIER_HIGH, HITL) в группе grants — тот же `setDomainRole` ядра.

Уровень NONE выдаётся через REST `POST /admin/v2-grants` и AI-тул `set_grant`.
Удаление кастомной роли снимает её назначения (сиром не остаётся).
Исключение пользователя из воркспейса отзывает доступ само (membership ушла —
гранты пустые), строка назначения остаётся инертной.

История объекта и откат снимка судятся той же суженной картой (#337):
поимённые NONE и доменные NONE закрывают их так же, как обычное чтение/запись.

## Обходные конструкции — доверенный круг (#274)

Создание/правка конструкций, чьё исполнение определено автором, а не ролью
исполнителя, — только owner/admin:
- computed-reqs (POST/DELETE), script-button config, http-button config — REST `requireRole('admin')`;
- AI-паритет: TOOL_MIN_ROLE create/update/delete_computed, configure_http_button, configure_script_button = admin;
- исполнение остаётся доступным (http-button run — editor), аудируется в `_v2_http_button_audit`;
- GET /config http-button отдаёт не-admin'у заголовки как «•••»;
- инвентаризация легаси-формул отчётов: `GET /api/v2/:db/reports/legacy-formulas` (admin); обходит до 500 отчётов (limit listReports).
script-button исполняется в браузере кликнувшего под его JWT — серверного
аудита быть не может; действия аудируются как обычные API-вызовы.
