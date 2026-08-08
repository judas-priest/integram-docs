# Module: workspaces

**Path:** `src/api/v2/modules/workspaces/`
**Files:** `router.js`, `service.js`, `workspace-clone.js`, `workspace-templates.js`, `workspace-members.js`, `workspace-invitations.js`, `workspace-bots.js`, `wizard-service.js`
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
| DELETE | `/workspaces/:slug` | Delete workspace (owner only) |

### Members

| Method | Path | Description |
|--------|------|-------------|
| GET | `/workspaces/:slug/members` | List members with roles |
| POST | `/workspaces/:slug/members` | Add member by email |
| PUT | `/workspaces/:slug/members/:id` | Update member role |
| DELETE | `/workspaces/:slug/members/:id` | Remove member |

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
| POST | `/workspaces/:slug/clone` | Clone workspace to a new slug (`include_documents`, `include_members`). Blocked for remote workspaces. |

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

## Clone (`workspace-clone.js`)

Creates a new workspace and copies all EAV schema, views, reports, automations, and optionally data rows. EAV IDs are remapped to avoid conflicts. Clone is blocked for workspaces with `remote_dsn` set.

## Remote DSN

Workspaces can store their EAV data on a remote PostgreSQL server instead of the local one. Managed via the `remote_dsn` JSONB column on `_v2_workspaces`.

| Method | Path | Description |
|--------|------|-------------|
| POST | `/workspaces/:slug/test-remote-dsn` | Test DSN connectivity to a remote server |
| PATCH | `/workspaces/:slug/remote-dsn` | Save `remote_dsn` with connectivity check |

**Behavior:**
- `PATCH /:slug/remote-dsn` validates connectivity before saving. On success, bootstraps the workspace schema on the remote server.
- `DELETE /workspaces/:slug` uses the workspace-specific pool to `DROP SCHEMA` on the correct (potentially remote) server.
- Clone is blocked for workspaces with `remote_dsn`.

## DB Tables (global public schema)

- `_v2_workspaces` — `id`, `name`, `slug`, `db_name`, `org_id`, `settings` (JSONB), `modules` (JSONB), `owner_id`, `remote_dsn` (JSONB, nullable), `is_template` (boolean), `created_at`
- `_v2_memberships` — `user_id`, `workspace_id`, `role_id`, `joined_at`
- `_v2_invitations` — invite tokens, email, role, expiry
- `_v2_service_keys` — `smk_...` tokens (hashed), workspace scope
- `_v2_workspace_templates` — template manifests (JSONB)
