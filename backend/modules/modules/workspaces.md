# Module: workspaces

**Path:** `src/api/v2/modules/workspaces/`
**Files:** `router.js`, `service.js`, `workspace-clone.js`, `workspace-carrier.js`, `workspace-purge.js`, `workspace-templates.js`, `workspace-members.js`, `workspace-invitations.js`, `workspace-bots.js`, `workspace-move.js`, `last-seen.js`, `wizard-service.js` (`ls` of the module directory, minus `__tests__/`)
**Base URL:** `/api/v2/workspaces/...`  (no `:db` — global)
**Auth:** JWT required for all endpoints.

## Purpose

Workspace lifecycle management: create, read, update, delete workspaces; manage members, invitations, bots (service accounts), service keys; clone workspaces; manage workspace templates.

## Endpoints

### Workspace CRUD

| Method | Path | Description |
|--------|------|-------------|
| GET | `/workspaces` | List workspaces current user belongs to. Superadmins see **all** workspaces (with `superadminAccess: true` flag for workspaces where they have no explicit membership) and get an implicit `admin` role for access control. |
| POST | `/workspaces` | Create workspace (`name`, `slug`, optional `template`/`templateId`, `org_id`, `seedData`, `entityLocalIds`, `sharedKag`) |
| GET | `/workspaces/:slug` | Get workspace details and settings |
| PUT | `/workspaces/:slug` | Update name, settings (modules, security, ai, agent, icon, description, credentialsTypeId), `is_template` |
| DELETE | `/workspaces/:slug` | Delete workspace (owner only). Drops the schema **and** purges everything the workspace owns outside it — see Delete below. |

### Members

| Method | Path | Description |
|--------|------|-------------|
| GET | `/workspaces/:slug/members` | List members with roles — see response shape below |
| POST | `/workspaces/:slug/members` | Add member by email |
| PUT | `/workspaces/:slug/members/:id` | Update member role |
| DELETE | `/workspaces/:slug/members/:id` | Remove member |

#### `GET /workspaces/:slug/members` — response shape

`listMembers` (`workspace-members.js`) joins `_v2_memberships` to `_v2_users`
`WHERE m.workspace_id = ?`, ordered by `m.created_at`. Each row:

| Field | Source | Notes |
|-------|--------|-------|
| `id` | `_v2_memberships.id` | membership row id, not user id |
| `userId` | `m.user_id` | |
| `role` | `m.role` | `owner` / `admin` / `editor` / `viewer` |
| `createdAt` | `m.created_at` | when the membership was made |
| `lastActive` | `m.last_seen_at` | **the "last visit" value** — see `last-seen.js` below |
| `email`, `name`, `username`, `avatarUrl` | `_v2_users` | |
| `isBot` | `u.is_bot` | service accounts |
| `userCreatedAt` | `u.created_at` | |

Two things that surprise readers:

- **`lastActive`, not `lastSeenAt`.** The REST name is `lastActive` (already read
  by `MemberDialog.vue`); MCP `list_members` exposes the same column as
  `lastSeenAt`. Both are read-only projections of one column and cannot diverge.
- **The owner is inserted if missing.** If no row matches the owner, `listMembers`
  creates the membership row (`ON CONFLICT DO NOTHING`) and unshifts a synthetic
  entry with `createdAt: null` and `lastActive: null`. The field set of that
  entry must stay identical to the main query's — a reader distinguishes an empty
  field from an absent one, and `v-if` on screen swallows both alike.

### Invitations

| Method | Path | Description |
|--------|------|-------------|
| GET | `/workspaces/:slug/invitations` | List pending invitations |
| POST | `/workspaces/:slug/invite` | Invite by email (sends invite email) |
| DELETE | `/workspaces/:slug/invitations/:id` | Revoke invitation |

### Bots and Service Keys

| Method | Path | Description |
|--------|------|-------------|
| GET | `/workspaces/:slug/bots` | List bots |
| POST | `/workspaces/:slug/bots` | Create bot (service account) |
| PUT | `/workspaces/:slug/bots/:botId/role` | Update bot role |
| DELETE | `/workspaces/:slug/bots/:botId` | Delete bot |
| GET | `/workspaces/:slug/bots/:botId/keys` | List service keys for bot |
| POST | `/workspaces/:slug/bots/:botId/keys` | Issue service key for bot (`smk_...`) |
| DELETE | `/workspaces/:slug/bots/:botId/keys/:keyId` | Revoke service key |

### Wizard

| Method | Path | Description |
|--------|------|-------------|
| POST | `/workspaces/wizard/analyze` | Analyze free-text business description → match template + extract entities/modules. Body: `{ description, teamSize?, hasClients? }`. Returns `{ templateId, templateSlug, templateName, entities, modules, confidence }`. |
| POST | `/workspaces/wizard/adapt` | Refine entities/modules via LLM based on user prompt. Body: `{ currentEntities, prompt, currentModules }`. Returns `{ entities, modules }`. |
| POST | `/workspaces/:slug/fill-connector-tokens` | Inject secret tokens into connector `{{VAR}}` placeholders after workspace creation. Body: `{ tokens: { VAR_NAME: value } }`. Used by `WizardStepConnectors` to fill API keys/credentials without them ever being stored in the template manifest. |

**Logic (`wizard-service.js`):**
- `analyzeDescription`: keyword scoring (KEYWORD_MAP) → deterministic match if score ≥ 1, LLM fallback if weak, last resort = first template
- `adaptEntities`: LLM refines entities + 3 modules (portal/documents/automations). Name-based localId matching preserves IDs for unchanged entities.

### Templates

| Method | Path | Description |
|--------|------|-------------|
| GET | `/workspaces/templates` | List available workspace templates (`?scope=`) |
| GET | `/workspaces/templates/:id` | Get template manifest |
| POST | `/workspaces/templates` | Save workspace as a template |
| PUT | `/workspaces/templates/:id` | Update template metadata |
| DELETE | `/workspaces/templates/:id` | Delete template |
| POST | `/workspaces/:slug/apply-template` | Apply a template to an existing workspace (`dryRun`, `seedData`) |

### Clone

| Method | Path | Description |
|--------|------|-------------|
| POST | `/workspaces/:slug/clone` | Clone workspace to a new slug (`include_documents`, `include_members`). Blocked for remote workspaces. Response names everything that did **not** travel — see Clone below. |

## Slug Rules

- 3–64 chars, lowercase, starts with letter
- Regex: `/^[a-z][a-z0-9_-]{1,62}[a-z0-9]$/`
- Reserved slugs blocked (`src/shared/reserved-slugs.js`)

## Credentials Table (`credentialsTypeId`)

`settings.credentialsTypeId` (Number|null) — designates a workspace table as the **credentials store** for Telegram automations. When set, `send_telegram` actions can use `token_source: "table"` + `credentialsObjectId` to read the bot token from an EAV field of the selected record at runtime (instead of hardcoding the token in the action config).

Set via `PUT /workspaces/:slug` with `body.settings.credentialsTypeId = <typeId>` (or `null` to clear). UI: workspace settings → "Интеграции" tab → "Таблица с учётными данными".

---

## Modules Config

Each workspace has a `settings.modules` JSONB field (inside the `settings` column) controlling which features are enabled. Updated via `PUT /workspaces/:slug` with `body.settings.modules`:

```json
{
  "documents": true, "dashboards": true, "ai": true,
  "reports": true, "metrics": true, "automations": true,
  "webhooks": true, "graph": true, "connectors": true,
  "forms": true, "portal": true, "codespace": false,
  "calls": false
}
```

### Settings Schema (notable fields)

- `settings.modules` — feature flags (see above)
- `settings.credentialsTypeId` — see Credentials Table section
- `settings.ai_onboarding_dismissed` — boolean, hides the AI onboarding prompt

## Workspace Templates (`workspace-templates.js`)

Templates are stored as JSON manifests capturing the full workspace schema (types, reqs, row rules, views, reports, seed data). On `createWorkspace` with `templateId`, the manifest is replayed with EAV ID remapping. `applyTemplateToWorkspace` supports `dryRun` mode.

### Built-in templates

17 templates in `backend/src/data/templates/workspaces/`:

| Slug | Name |
|------|------|
| `ecommerce` | Интернет-магазин |
| `farmshop` | Фермерский магазин (с UDS-коннектором) |
| `manufacturing` | Производство |
| `beekeeping` | Пасека |
| `real-estate` | Недвижимость |
| `services-crm` | Сервисная компания / CRM |
| `restaurant` | Ресторан |
| `education` | Образование |
| `freelance` | Фриланс |
| `ad-agency` | Рекламное агентство |
| `logistics` | Логистика |
| `clinic` | Клиника |
| `auto-service` | Автосервис |
| `legal` | Юридическая фирма |
| `events` | Мероприятия |
| `perelidos-accelerator` | Акселератор Перелидос |
| `meta-kb-archipelago` | Meta KB Архипелаг (НИОКР) |

The `farmshop` template includes two UDS API connectors and a cron automation (`*/5 * * * *`) for automatic incremental sync. Connector auth tokens use `{{VAR}}` placeholders filled via `fill-connector-tokens` after creation.

### Template manifest — agents

**manifest.agents** — array of agent definitions. Each entry has `slug`, `name`, `systemPrompt`, `model`, `tools`, etc. On apply, agents are inserted into `_v2_agent_registry` with `ON CONFLICT (slug) DO NOTHING`. Used by `meta-kb-archipelago` template (11 НИОКР analysis agents).

### Template manifest — connectors and automations

**manifest.connectors** — array of connector definitions. Auth/params fields may contain `{{VAR}}` interpolation placeholders filled post-creation. Each connector has a `localId` used for cross-references.

**manifest.automations** — automation actions of type `run_connector` reference connectors via `connectorLocalId`. On apply, `workspace-templates.js` resolves `connectorLocalId` → real connector ID in the created workspace.

### Template manifest — reports, timeseries

**manifest.reports** — array of report definitions. Each entry:
- `name` — report display name
- `icon` — emoji or null
- `parentTypeLocalId` — local ID of the source table
- `where` — SQL WHERE clause or null
- `storedLimit` — default row limit or null
- `columns[]` — `{ colLocalId, func, displayName, totalFunc, storedFrom, storedTo, hidden }`

**manifest.timeseries** — array of timeseries source placeholders. Each entry:
- `source_id` — numeric BIGINT source ID
- `metric` — metric name
- `unit` — unit label or null
- `description` — description or null

On apply: placeholder rows ingested so sources appear in the timeseries list.
Existing templates without these fields apply correctly (empty arrays).

## Last seen (`last-seen.js`)

"When was this member last in this workspace" — a **column**, not a log:
`_v2_memberships.last_seen_at`. See
[ADR-026](../../../docs/adr/026-presence-and-last-seen.md); the presence log
(`_v2_presence_log`) answers a different question and must not be used for this.

`touchLastSeen(pool, workspaceId, userId)` is the column's only writer. It is
called from `workspaceRoleMiddleware` (`middleware/jwt-auth.js`), i.e. on every
`/:db` request.

**Throttle: 5 minutes.** One UPDATE per (user, workspace) pair per
`THROTTLE_MS = 5 * 60 * 1000`. Without it this would be an UPDATE on *every*
request; PostgreSQL does not edit a row in place but writes a new version and
leaves a dead one, so the membership table — read on every request — and its
indexes would bloat faster than autovacuum keeps up. Five minutes buys a UI
precision of five minutes at no more than 12 writes per hour per member.

Throttle state is a plain in-process `Map` keyed `${userId}:${workspaceId}`, not
Redis: HTTP is served by one process, a restart costs one extra UPDATE per user,
and a Redis round-trip would cost a network call per request. Capped at 5000 keys
with a 1000-key eviction batch (bots that roam many workspaces). `_resetLastSeenCache()`
is exported for tests only.

**The function THROWS — call it only through `sideEffect`.**

```js
sideEffect('ws_last_seen', touchLastSeen(pool, workspaceId, userId));
```

This is deliberate and stated in its JSDoc, against the convention of its
neighbours whose "never throws" is a literal promise backed by a test. The
throttle decision itself cannot fail, but a failing UPDATE escapes as a rejected
promise. Measured by injecting a failure into `execSql`: **a bare `await` in
`workspaceRoleMiddleware` turns a legitimately allowed request into `403
"Workspace access denied"`, not a 500** — the middleware's blanket `catch` reads
a failed auxiliary write as a failed access check. Guard:
`middleware/__tests__/last-seen-touch.test.js`.

**Where the mark is written and where it is not.** `workspaceRoleMiddleware` calls
`touch()` on its three success branches (superadmin bypass, role from cache, role
resolved from DB) and on none of its four refusal paths — a denied request is not
a visit.

**Not everyone with access gets a mark, and it barely shows.** `resolveEffectiveRole`
grants access along three paths where a membership row with the right
`workspace_id` may not exist at all:

| Case | Why the UPDATE hits zero rows |
|------|-------------------------------|
| Owner by `workspace.owner_id` | may have no membership row |
| Member via organization | their row has `workspace_id` NULL, `org_id` set |
| Superadmin | no membership row anywhere |

The UPDATE matches nothing and says nothing. On screen the consequence is almost
invisible, because `listMembers` selects with the same predicate
`WHERE m.workspace_id = ?` — exactly the set the UPDATE writes to. Org-members and
superadmins never appear in the list at all. The owner is the one leftover case,
and it self-heals: `listMembers` creates their row on first open, and marks land
from then on.

**Storage note.** `ensureRegistryTables` runs `ALTER TABLE _v2_memberships SET
(fillfactor = 90)` on every start (idempotent, takes only
`ShareUpdateExclusiveLock`, does not conflict with reads/writes). The spare room
gives the frequent `last_seen_at` write a chance to be a HOT update, skipping
indexes and leaving fewer dead tuples. **There is deliberately no index on
`last_seen_at`** — an index on it would cancel exactly that HOT update. It only
affects pages filled after the command runs; already-full pages are rewritten
only by `VACUUM FULL`, which is not performed.

## Move to another server (`workspace-move.js`)

Guards `PATCH /workspaces/:slug/remote-dsn`. The move writes a connection string
and creates an empty schema on the far side — **records are not carried over.**
They stay in the old database: access to them is lost, the records themselves are
not; clearing `remote_dsn` brings everything back.

The danger is that the disconnection is silent — the workspace opens, tables are
there, rows are gone, and an empty table is indistinguishable from one nobody ever
filled. So the silence is replaced by a refusal: a move from a non-empty source to
an empty target fails with `MOVE_WOULD_STRAND_DATA` until the caller confirms it
explicitly. Checks run **before any change**: registering the target first would
make `getPoolForDb` resolve the source check to the new address.

## Carry registry (`src/api/v2/registry/workspace-carry.js`)

One declaration of what a workspace owns and what happens to it. Every table gets
an entry with: **kind** (`настройка` / `содержимое` / `журнал` / `секрет` /
`выводимое`), **home** (`ws` — workspace schema, `system` — global DB with an
ownership column, `graph` — global DB but carried by a dedicated function, `disk`),
**clone mode** (`copy` / `regenerate` / `custom` / `skip`) and **template mode**
(`manifest` / `skip`), plus a mandatory `why`.

Columns are never listed in code — they are read from `information_schema` of the
source (`carryColumns`, `workspace-carrier.js:38`, used by `copyTable`, `:177`).
Three guards keep the registry
from falling behind (`src/api/v2/registry/__tests__/`): `carry-ddl-files.test.js`
(a new file with `CREATE TABLE IF NOT EXISTS` must be declared or exempted with a
reason), `carry-tables.test.js` (a new table inside an already declared module —
names collected by *executing* the declared `ensure*` on a recording pool),
`carry-invariants.test.js` (entry consistency: журнал/выводимое must be `skip`,
`custom` must name an existing function, and so on).

Ownership — a separate question from carriability — is declared in
`src/api/v2/registry/workspace-ownership.js` and used by delete/rollback purge and
by the APM diagnostic snapshot. Rationale: `docs/adr/026-workspace-carry-registry.md`.

## Clone (`workspace-clone.js`, `workspace-carrier.js`)

`workspace-clone.js` holds the phase order and the phases themselves;
`workspace-carrier.js` holds the per-table carry mechanics (`carryColumns`,
`primaryKeyColumns`, `advanceIdentity`, `copyTable`, `copyScoped`, `stripCarried`,
`carryPlan`, `insertMapped`, `loadRows`, `rowFailures`). The phase functions named
by the registry (`cloneDocuments`, `cloneCodespace`, `cloneRowRules`,
`cloneAgentMemory`) stay in `workspace-clone.js` — the registry guards read *their
text* at that path.

Creates a new workspace and copies, per the carry registry: the EAV data table,
the satellite tables of the schema (`home: 'ws'`), the property that lives outside
the schema (portal config, dashboards, `kag_*`, shared state, graph, agent memory),
and the files on disk. Documents and members travel on the `include_documents` /
`include_members` flags.

**EAV IDs are preserved, not remapped.** The data table is copied with
`OVERRIDING SYSTEM VALUE` (`workspace-clone.js:582`), and satellite rows keep
their own ids the same way (`copyTable`, `workspace-carrier.js:231`): satellites reference each
other by id (`_v2_git_pull_requests.repo_id` → `_v2_git_repos.id`,
`_v2_webhook_deliveries.webhook_id` → `_v2_webhooks.id`), and re-issuing ids would
break those links silently. ID remapping is a property of the **template** path,
not the clone path (`idMap`, `mapPortalIds` in `workspace-templates.js`).

Response: `{ id, slug, name, dbName, orgId, clonedFrom, notCarried[],
carryFailed[], carryPartial[] }`.

- `notCarried` — the **complete** list of what is not carried by design, each entry
  `{table, kind, home, why}` (registry entries with `clone.mode: 'skip'`);
- `carryFailed` — what was supposed to travel and did not;
- `carryPartial` — what travelled incompletely (columns missing in the target,
  column names unusable in SQL, sequence not advanced).

On failure the clone rolls back in reverse order: cleanup steps drop the new
schema, delete the workspace row and memberships, and run `purgeOutsideSchema` on
the half-built clone. Clone is blocked for workspaces with `remote_dsn` set.

## Delete and purge (`workspace-purge.js`)

`DROP SCHEMA … CASCADE` removes exactly what lives in the schema. Portal config,
dashboards, forms, `kag_*` knowledge, shared state, graph and agent memory live in
the global DB; bare git repos and uploaded files live on disk. All of it survives
the schema drop — and since the DB name is derived from the slug, the *next*
workspace with the same slug would inherit it.

`purgeOutsideSchema(pool, dbName, { wsId, serverRoot })`:

- asks the **DB catalog** (`information_schema.columns`), not a list in code, for
  every global-DB table that has an ownership column (`OWNERSHIP_COLUMNS` in
  `registry/workspace-ownership.js`), skipping declared exceptions (`NOT_OWNED`);
- decides what to substitute by the **column type**, not its name — `workspace_id`
  is INTEGER in `_v2_dlp_rules` but TEXT holding the DB name in `llm_calls` and
  five more. Live count (2026-08-20): eleven tables carry `workspace_id`, five
  numeric and six TEXT; the count is never written down in prose — it is taken
  from the catalog by the guard `carry-ownership.test.js`. An unrecognised type is
  reported, never guessed — `citext` and other extension types are resolved
  through `udt_name`, since `data_type` only says `USER-DEFINED`;
- removes on-disk directories: codespace bare repos (`GIT_ROOT/<db>`), and
  `download/<db>`, `templates/custom/<db>` under `INTEGRAM_SERVER`;
- never throws — it is called from rollback paths. It returns a three-part report
  `{ problems, unknownKind, skipped }`, and the split matters: `problems` means
  *this workspace's* property was left behind, `unknownKind` is a property of the
  DB (a stray column of an unknown type shows up when purging **any** workspace),
  and `skipped` names deliberate skips (a numeric column while the workspace id
  has not been issued yet). Deciding that a failed purge is a failed delete
  belongs to the caller.

Order in `deleteWorkspace` (`service.js:395`): `DROP SCHEMA` on the workspace's own
pool → `purgeOutsideSchema` → delete memberships/invitations → delete the
`_v2_workspaces` row. The purge must run **before** the workspace row is deleted:
`_v2_dlp_rules.workspace_id` references it `ON DELETE CASCADE`.

**A non-empty `problems` aborts the delete** with
`AppError(WORKSPACE_PURGE_INCOMPLETE, …, 500)`, and the `_v2_workspaces` row is
deliberately kept. Previously the problems were only logged and the row was
deleted regardless: measured live with a `BEFORE DELETE` trigger, the answer was
`{deleted: true}` while the rows survived and the slug was freed — so the next
workspace with that slug would have inherited them. Keeping the row keeps the slug
taken, which leaves the leftovers without an heir; the delete can be retried once
the obstacle is gone. `unknownKind` and `skipped` are logged and never abort.

## Remote DSN

Workspaces can store their EAV data on a remote PostgreSQL server instead of the local one. Managed via the `remote_dsn` JSONB column on `_v2_workspaces`.

| Method | Path | Description |
|--------|------|-------------|
| POST | `/workspaces/:slug/test-remote-dsn` | Test DSN connectivity to a remote server |
| PATCH | `/workspaces/:slug/remote-dsn` | Save `remote_dsn` with connectivity check |

**Behavior:**
- `PATCH /:slug/remote-dsn` validates connectivity before saving. On success, bootstraps the workspace schema on the remote server.
- The move does **not** carry data: rows stay in the previous DB. `assertMoveSafe` (`workspace-move.js`) runs before any change and refuses with `MOVE_WOULD_STRAND_DATA` (409) when the source holds rows and the target is empty, unless the body passes `confirmEmptyTarget: true`.
- `DELETE /workspaces/:slug` uses the workspace-specific pool to `DROP SCHEMA` on the correct (potentially remote) server; the outside-the-schema purge always runs on the main pool (see Delete and purge).
- Clone is blocked for workspaces with `remote_dsn`.

## DB Tables (global public schema)

DDL lives in `src/api/v2/registry/tables.js` (except `_v2_service_keys`, see below).

**`_v2_workspaces`** — `id`, `slug`, `name`, `db_name` (`VARCHAR(15)`), `org_id`,
`owner_id`, `template`, `is_template` (bool), `settings` (JSONB), `remote_dsn`
(JSONB, nullable), `created_at`. Unique on `slug` and on `db_name`.

> There is **no `modules` column.** Feature flags live inside the `settings`
> JSONB as `settings.modules` (see Modules Config above).

**`_v2_memberships`** —

| Column | Type | Notes |
|--------|------|-------|
| `id` | `BIGINT` identity | PK; this is what `GET .../members` returns as `id` |
| `user_id` | `BIGINT NOT NULL` | |
| `org_id` | `BIGINT` nullable | set instead of `workspace_id` for org-wide membership |
| `workspace_id` | `BIGINT` nullable | NULL for org-wide membership |
| `role` | `TEXT NOT NULL DEFAULT 'viewer'` | `CHECK (role IN ('owner','admin','editor','viewer'))` |
| `last_seen_at` | `TIMESTAMPTZ` nullable | last visit; see `last-seen.js` above. No index — deliberately |
| `created_at` | `TIMESTAMPTZ NOT NULL` | |

`UNIQUE (user_id, org_id, workspace_id)`; table storage parameter `fillfactor = 90`.

> There is no `role_id` and no `joined_at` — the columns are `role` and
> `created_at`. `last_seen_at` was added by the migration list at the bottom of
> `registry/tables.js`, so old installations get it via
> `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`.

**`_v2_invitations`** — `email`, `inviter_id`, `org_id`, `workspace_id`, `role`
(no `owner`), `token` (unique), `status` (`pending`/`accepted`/`expired`/`revoked`),
`expires_at`, `created_at`.

**`_v2_workspace_templates`** — `name`, `slug` (unique), `description`, `icon`,
`visibility` (`private`/`org`/`public`/`system`), `scope`, `category`, `org_id`,
`owner_id`, `manifest` (JSONB), `version`, timestamps.

**`_v2_service_keys`** — `smk_...` tokens. DDL in
`backend/scripts/migrate-service-accounts.sql`, not in `registry/tables.js`.
Columns: `user_id` (→ a bot user in `_v2_users`, `is_bot = true`), `key_hash`
(`CHAR(64)`, SHA-256 of the raw token, unique), `key_prefix` (`VARCHAR(12)`, first
12 chars for display), `name`, `created_by` (the human who issued it), `created_at`,
`last_used_at`, `expires_at` (NULL = no expiry), `revoked_at` (soft delete).
**Scoping is by bot user, not by workspace** — the workspace binding comes from
the bot's membership row.

DDL for all of these lives in `src/api/v2/registry/tables.js` (except `_v2_service_keys`, see below).

- `_v2_workspaces` — `id`, `slug`, `name`, `db_name`, `org_id`, `owner_id`, `template`, `is_template` (boolean), `settings` (JSONB), `remote_dsn` (JSONB, nullable), `created_at`; UNIQUE on `slug` and on `db_name`. There is **no** `modules` column — the feature flags live inside `settings.modules` (`tables.js:33`)
- `_v2_memberships` — `id`, `user_id`, `org_id`, `workspace_id`, `role` (TEXT, one of `owner`/`admin`/`editor`/`viewer`), `created_at` (`tables.js:49`). The role is stored as text — there is no `role_id` and no `joined_at`. The table-level `UNIQUE (user_id, org_id, workspace_id)` catches nothing: both ownership columns are nullable and two NULLs are never equal in Postgres. Real uniqueness comes from two partial indexes built by `MEMBERSHIP_UNIQUE` (`tables.js:247`): `uq_membership_workspace` on `(user_id, workspace_id) WHERE workspace_id IS NOT NULL` and `uq_membership_org` on `(user_id, org_id) WHERE org_id IS NOT NULL`. `ON CONFLICT` must target one of those pairs, not the triple
- `_v2_invitations` — `email`, `inviter_id`, `org_id`, `workspace_id`, `role`, `token` (UNIQUE), `status` (`pending`/`accepted`/`expired`/`revoked`), `expires_at`, `created_at`
- `_v2_service_keys` — `smk_...` tokens (SHA-256 hash + 12-char prefix), attached to a **bot user** (`user_id`), not to a workspace: the workspace scope comes from that bot's membership. DDL: `backend/scripts/migrate-service-accounts.sql`
- `_v2_workspace_templates` — template manifests (JSONB), `slug` UNIQUE, `visibility` (`private`/`org`/`public`/`system`), `scope`, `category`, `version`
