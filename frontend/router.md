# Router

**Файл:** `src/router/index.js`

## Структура маршрутов

Маршруты делятся на три группы:

### 1. Публичные (без layout, без auth)

| Path | Name | View |
|------|------|------|
| `/` | `home` | `Landing.vue` |
| `/login` | `login` | `auth/Login.vue` |
| `/register` | `register` | `auth/Register.vue` |
| `/forgot-password` | `forgot-password` | `auth/ForgotPassword.vue` |
| `/reset-confirm` | `reset-confirm` | `auth/ResetConfirm.vue` |
| `/otp` | `otp` | `auth/OtpVerify.vue` |
| `/forms/:token` | `form-public` | `forms/FormPublic.vue` |
| `/public/docs/:db/:id` | `public-doc` | `documents/PublicDocument.vue` |
| `/public/records/:token` | `public-record` | `data/PublicRecord.vue` |
| `/public/v/:token` | `public-view` | `data/PublicTableView.vue` |
| `/auto-login/:token` | `auto-login` | `auth/AutoLogin.vue` |
| `/access-denied` | `access-denied` | `auth/AccessDenied.vue` |
| `/error` | `error` | `auth/Error.vue` |
| `/:pathMatch(.*)*` | `not-found` | `auth/NotFound.vue` |

### 2. Глобальные (с layout, requiresGlobalAuth)

| Path | Name | View |
|------|------|------|
| `/desktop` | `desktop` | `desktop/DesktopView.vue` |
| `/testing` | `testing-global` | `testing/TestingView.vue` |
| `/workspaces` | `workspaces` | `workspaces/WorkspaceList.vue` |
| `/workspaces/new` | `workspace-create` | `workspaces/WorkspaceCreate.vue` |
| `/workspaces/:slug` | `workspace-edit` | `workspaces/WorkspaceEdit.vue` |
| `/workspaces/:slug/members` | `workspace-members` | `workspaces/WorkspaceMembers.vue` |
| `/orgs` | `orgs` | `orgs/OrgList.vue` |
| `/orgs/:slug` | `org-edit` | `orgs/OrgEdit.vue` |
| `/orgs/:slug/members` | `org-members` | `orgs/OrgMembers.vue` |

### 3. Workspace-scoped (/:db/, requiresAuth)

Все маршруты вложены под `/:db/` и используют `AppLayout`.

| Path | Name | View | Модуль |
|------|------|------|--------|
| `/:db/` | `dashboard` | `Dashboard.vue` | — |
| `/:db/tables` | `data-home` | `data/DataHome.vue` | — |
| `/:db/tables/:typeId` | `object-list` | `data/ObjectList.vue` | — |
| `/:db/tables/:typeId/:id` | `object-view` | `data/ObjectView.vue` | — |
| `/:db/tables/:typeId/:id/edit` | `object-edit` | `data/ObjectEdit.vue` | — |
| `/:db/tables/:typeId/new` | `object-create` | `data/ObjectEdit.vue` | — |
| `/:db/tables/:typeId/import` | `object-import` | `data/ObjectImport.vue` | — |
| `/:db/trash` | `trash` | `data/TrashView.vue` | — |
| `/:db/schema` | `schema` | `schema/TypeList.vue` | — |
| `/:db/schema/new` | `type-create` | `schema/TypeCreate.vue` | — |
| `/:db/schema/:typeId` | `type-edit` | `schema/TypeEdit.vue` | — |
| `/:db/reports` | `reports` | `reports/ReportList.vue` | reports |
| `/:db/reports/:reportId` | `report-view` | `reports/ReportView.vue` | reports |
| `/:db/reports/:reportId/edit` | `report-edit` | `reports/ReportEdit.vue` | reports |
| `/:db/metrics` | `metrics` | `timeseries/TimeseriesView.vue` | — |
| `/:db/dashboards` | `dashboards-list` | `dashboards/DashboardList.vue` | dashboards |
| `/:db/dashboards/new` | `dashboard-create` | `dashboards/DashboardBuilder.vue` | dashboards |
| `/:db/dashboards/:dashboardId` | `dashboard-view` | `dashboards/DashboardBuilder.vue` | dashboards |
| `/:db/files` | `files` | `files/FileManager.vue` | — |
| `/:db/normalizer/new` | `normalizer-new` | `normalizer/NormalizerWizard.vue` | — |
| `/:db/normalizer/:jobId` | `normalizer-status` | `normalizer/NormalizerWizard.vue` | — |
| `/:db/normalizer/:jobId/review` | `normalizer-review` | `normalizer/NormalizerReview.vue` | — |
| `/:db/teamchat` | `teamchat` | `teamchat/TeamChatLayout.vue` | teamchat |
| `/:db/teamchat/room/:roomId` | `teamchat-room` | `teamchat/TeamChatLayout.vue` | teamchat |
| `/:db/codespace` | `codespace-list` | `codespace/CodespaceList.vue` | codespace |
| `/:db/codespace/:slug` | `codespace-repo` | `codespace/RepoView.vue` | codespace |
| `/:db/decisions` | `decisions` | `decisions/DecisionList.vue` | decisions |
| `/:db/decisions/search` | `decisions-search` | `decisions/SearchWithAnamnesis.vue` | decisions |
| `/:db/decisions/:id` | `decision-detail` | `decisions/DecisionDetail.vue` | decisions |
| `/:db/documents` | `documents` | `documents/DocumentList.vue` | documents |
| `/:db/documents/new` | `document-create` | `documents/DocumentEditor.vue` | documents |
| `/:db/documents/:docId` | `document-edit` | `documents/DocumentEditor.vue` | documents |
| `/:db/documents/:docId/versions` | `document-versions` | `documents/DocumentVersions.vue` | documents |
| `/:db/documents/:docId/sharing` | `document-sharing` | `documents/DocumentSharing.vue` | documents |
| `/:db/documents/trash` | *(redirect)* | → `/:db/trash?section=docs` | — |
| `/:db/forms` | `forms` | `forms/FormList.vue` | forms |
| `/:db/forms/new` | `form-create` | `forms/FormBuilder.vue` | forms |
| `/:db/forms/:token` | `form-edit` | `forms/FormBuilder.vue` | forms |
| `/:db/automations` | `automations` | `automations/AutomationsDashboard.vue` | automations |
| `/:db/automations/list` | `automations-list` | `automations/AutomationList.vue` | automations |
| `/:db/automations/new` | `automation-create` | `automations/AutomationEdit.vue` | automations |
| `/:db/automations/:id` | `automation-edit` | `automations/AutomationEdit.vue` | automations |
| `/:db/automations/:id/runs` | `automation-runs` | `automations/AutomationRuns.vue` | automations |
| `/:db/webhooks` | `webhooks` | `webhooks/WebhookList.vue` | webhooks |
| `/:db/webhooks/new` | `webhook-create` | `webhooks/WebhookEdit.vue` | webhooks |
| `/:db/webhooks/:id` | `webhook-edit` | `webhooks/WebhookEdit.vue` | webhooks |
| `/:db/connectors` | `connectors` | `connectors/ConnectorList.vue` | connectors |
| `/:db/connectors/:id` | `connector-edit` | `connectors/ConnectorEdit.vue` | connectors |
| `/:db/connectors/:id/run` | `connector-run` | `connectors/ConnectorRun.vue` | connectors |
| `/:db/graph` | `graph` | `graph/GraphExplorer.vue` | graph |
| `/:db/graph/path` | `graph-path` | `graph/GraphPathFinder.vue` | graph |
| `/:db/ai` | `ai-chat` | `ai/AiChat.vue` | ai |
| `/:db/ai/fill/:typeId` | `ai-fill` | `ai/AiFillTable.vue` | ai |
| `/:db/ai/column-config/:typeId/:reqId` | `ai-column-config` | `ai/AiColumnConfig.vue` | ai |
| `/:db/notifications` | `notifications` | `notifications/NotificationList.vue` | — |
| `/:db/admin/agents` | `admin-agents` | `agents/AgentList.vue` | — |
| `/:db/admin` | `admin` | `admin/AdminDashboard.vue` | — |
| `/:db/admin/v2-grants` | `admin-v2-grants` | `admin/V2Grants.vue` | — |
| `/:db/admin/roles` | `admin-roles` | `admin/Roles.vue` | — |
| `/:db/admin/row-rules` | `admin-row-rules` | `admin/RowRules.vue` | — |
| `/:db/admin/backup` | `admin-backup` | `admin/Backup.vue` | — |
| `/:db/admin/export` | `admin-export` | `admin/Export.vue` | — |
| `/:db/admin/dlq` | `admin-dlq` | `admin/DeadLetterQueue.vue` | — |
| `/:db/admin/audit` | `admin-audit` | `admin/AuditLog.vue` | — |
| `/:db/admin/qa-bugs` | `admin-qa-bugs` | `admin/QaBugs.vue` | — |
| `/:db/admin/qa-results` | `admin-qa-results` | `admin/QaResults.vue` | — |
| `/:db/conference` | `conference` | `calls/ConferenceView.vue` | — |
| `/:db/teamchat/tasks` | `teamchat-tasks` | `teamchat/TaskList.vue` | — |
| `/:db/teamchat/board/:roomId` | `teamchat-board` | `teamchat/TaskBoard.vue` | — |
| `/:db/portal-config` | *(redirect)* | → `/:db/portal-editor` (алиас обратной совместимости) | — |
| `/:db/portal-editor` | `portal-editor` | `portal/PortalVisualEditor.vue` | — |
| `/:db/testing` | `testing` | `testing/TestingView.vue` | — |
| `/:db/settings` | `workspace-settings` | `workspaces/WorkspaceEdit.vue` | — |
| `/:db/settings/members` | `workspace-settings-members` | `workspaces/WorkspaceMembers.vue` | — |
| `/:db/settings/profile` | `profile` | `settings/Profile.vue` | — |
| `/:db/settings/password` | `change-password` | `settings/ChangePassword.vue` | — |
| `/:db/settings/api-keys` | `api-keys` | `settings/ApiKeys.vue` | — |

## Guards

**`setupRouterGuards(router)`** — настраивает `router.beforeEach`:

1. Публичные маршруты (`PUBLIC_ROUTES`) — пропускаются без проверки
2. `requiresAuth` — проверяет `globalToken`, при отсутствии → `/login?db=...`
3. **Module guard** — если маршрут в `ROUTE_MODULE_MAP`, проверяет `workspace.moduleEnabled(moduleName)`. При отключённом модуле → `dashboard`
4. **Admin guard** — если маршрут имеет `meta.requiresAdmin`, проверяет `workspace.isAdmin() || workspace.isOwner()`. При отказе → `dashboard`
5. `requiresGlobalAuth` — проверяет `globalToken` без привязки к workspace

**Валидация `:db`** — регулярное выражение `DB_NAME_RE`: буква в начале, только `[a-zA-Z0-9_-]`, максимум 64 символа.
