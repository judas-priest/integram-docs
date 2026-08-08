# Services — API-клиенты

**Директория:** `src/services/`

Каждый файл — тонкая обёртка вокруг `services/api.js`. Базовый URL для workspace-scoped запросов: `/api/v2/{db}/...`.

## api.js (базовый клиент)

Экспортирует фабрики `getDbApi(db)` и `getGlobalApi()`. Каждый инстанс имеет:
- Автоматическую подстановку `Authorization: Bearer {token}` из `stores/auth`
- Перехватчик ответа: при 401 пробует silent refresh через cookie (`iamRefresh`), затем повторяет запрос. При повторном 401 вызывает `onUnauthorized` (редирект на `/login`)
- Автоматический retry при 5xx (до 3 раз с экспоненциальным backoff)
- При 403 "Workspace access denied" — редирект на `/access-denied`
- `sanitizeApiError(error)` — очищает сообщения об ошибках от внутренних деталей (SQL, пути, стектрейсы)
- `getAuthToken()` — получить текущий JWT (для fetch-based SSE запросов)

## Список сервисов

| Файл | Backend-модуль | Базовый путь |
|------|---------------|-------------|
| `iam.js` | iam | `/api/v2/iam/` |
| `workspaces.js` | workspaces | `/api/v2/workspaces/` — includes `wizardAnalyze(data)` → `POST /wizard/analyze`, `wizardAdapt(data)` → `POST /wizard/adapt`, `fillConnectorTokens(slug, tokens)` → `POST /:slug/fill-connector-tokens` |
| `orgs.js` | orgs | `/api/v2/orgs/` |
| `objects.js` | objects | `/api/v2/:db/objects/` |
| `schema.js` | schema | `/api/v2/:db/schema/` — methods: `getSnapshots`, `getSnapshot`, `rebuildFlatViews` |
| `lookups.js` | lookups | `/api/v2/:db/lookups/` |
| `reports.js` | reports | `/api/v2/:db/reports/` — methods: `history` (audit log), `stream` (SSE EventSource for large reports) |
| `dashboards.js` | dashboards | `/api/v2/:db/dashboards/` |
| `documents.js` | documents | `/api/v2/:db/documents/` |
| `files.js` | files | `/api/v2/:db/files/` |
| `automations.js` | automations | `/api/v2/:db/automations/` — methods: `list`, `get`, `create`, `update`, `remove`, `run`, `runs`, `seedSystem`, `runBatch`, `batchStatus` |
| `webhooks.js` | webhooks | `/api/v2/:db/webhooks/` — methods: `deliveries`, `retryDelivery` |
| `connectors.js` | connectors | `/api/v2/:db/connectors/` |
| `httpButton.js` | http-button | `/api/v2/:db/http-button/` | `getConfig`, `saveConfig`, `run` — HTTP Button column (type 1016) |
| `scriptButton.js` | script-button | `/api/v2/:db/script-button/` | `getConfig`, `saveConfig`, `fetchProxy` — Script Button column (type 1020); `fetchProxy` proxies HTTP from browser Web Worker scripts (CORS bypass, SSRF-protected) |
| `forms.js` | forms | `/api/v2/:db/forms/` |
| `comments.js` | comments | `/api/v2/:db/comments/` |
| `notifications.js` | notifications | `/api/v2/:db/notifications/` |
| `graph.js` | graph | `/api/v2/:db/graph/` |
| `ai.js` | ai | `/api/v2/:db/ai/` | `runButton` uses 180s timeout (agent mode with web_search can take 60-120s); `getTraces(db, threadId)` — GET traces for a thread; `uploadFile(db, file)` — POST file upload for AI context |
| `timeseries.js` | timeseries | `/api/v2/:db/timeseries/` |
| `admin.js` | admin | `/api/v2/:db/admin/` |
| `codespace.js` | codespace | `/api/v2/:db/codespace/` | `commitFile`, `commitMultiFiles(db, slug, {branch, files, message})` — атомарный коммит нескольких файлов; `getGithubSync`, `configureGithubSync`, `removeGithubSync`, `pushToGithub`, `pullFromGithub` — GitHub sync operations |
| `normalizer.js` | normalizer | `/api/v2/:db/normalize/` |
| `agentRegistry.js` | agent-registry | `/api/v2/:db/agents/` |
| `agentSuggestions.js` | agent-suggestions | `/api/v2/:db/agent-suggestions/` | `list`, `get`, `apply`, `confirmApply`, `dismiss`, `whyNot`, `searchSimilar`, `telemetry` |
| `portal.js` | portal | `/api/v2/:db/portal/api/` |
| `views.js` | views | `/api/v2/:db/views/` |
| `telegramBots.js` | telegramBots | `/api/v2/:db/portal/api/bots/` — list(db), get(db, id), create(db, data), update(db, id, data), remove(db, id), sync(db, id), status(db, id), test(db, id, data), automations(db, botId) |
| `testing.js` | testing | `/api/v2/:db/testing/sessions` |
| `trash.js` | objects (trash) | `/api/v2/:db/objects/trash` — корзина: удалённые записи, восстановление |
| `desktop.js` | desktop | `/api/v2/desktop/` |
| `calls.js` | calls | `/api/v2/:db/calls/` — `history(db, {limit})` → `GET /history`; `turnCredentials(db)` → `GET /turn-credentials` (short-lived HMAC-SHA1 TURN creds) |
| `teamchatService.js` | teamchat | `/api/v2/:db/teamchat/` — `listRooms`, `createRoom`, `getRoom`, `updateRoom`, `deleteRoom`; `listTopics`, `createTopic`, `getTopic`, `updateTopic`, `deleteTopic`; `listAllTopics`, `attachTopic`, `detachTopic`, `summarizeTopic`, `moveMessage`, `moveMessagesBulk`, `moveTopic`; `listMessages`, `createMessage`, `updateMessage`, `deleteMessage`, `searchMessages`, `loadOlderMessages`; `listMembers`, `addMember`, `removeMember`, `updateMemberRole`; `listRecentTopics`, `muteTopic`, `unmuteTopic`, `listMutedTopics`, `markTopicRead`; `pinTopic`, `unpinTopic`, `makeTask`, `unmakeTask`; `getKanban`, `getTasks`; `getTopicLink`, `getMessageLink`; `listPublicRooms`, `joinRoom`; `exportTopicToDocument`, `refreshDocument`; `getWMatrix(db, {roomId?, days?})` |
| `decisionsService.js` | decisions | `/api/v2/:db/decisions/` — `list`, `create`, `get`, `update`, `delete`; `listLinks`, `createLink`, `deleteLink`; `getDiscussions`; `getConflicts(db, id)`, `getHistory(db, id)`, `getKagStats(db, id)` |
| `metaKbService.js` | meta-kb | `/api/v2/:db/meta-kb/` — `listTopics(db)`, `getTopicDebate(db, topicId)`, `reviewChange(db, id, data)`, `getAnalytics(db)`, `exportDebate(db, debateId)`, `exportDebateDocx(db, debateId)`, `listRules(db)`, `createRule(db, data)`, `runRules(db, entityIds)`, `getWelcome(db)`, `listChanges(db, status, limit)`, `proposeChange(db, data)`, `listSnapshots(db, limit)`, `createSnapshot(db, label)`, `diffSnapshots(db, a, b)`, `deleteRule(db, id)` |
| `dlpService.js` | ai (DLP) | `/api/v2/:db/ai/dlp/rules` — `list`, `create`, `update`, `remove`; also `getWorkspace(slug)`, `updateWorkspaceSettings(slug, settings)` for closed_contour and dlp.enabled |
| `sandboxService.js` | teamchat | `/api/v2/:db/teamchat/topics/:id/messages` — code cells в топиках: создание, выполнение |
| `graphVisualizationService.js` | — | D3-based граф: layout, rendering (клиентский, без HTTP) |
| `ttsService.js` | tts | `/api/v2/:db/tts/` — `synthesize`, `voices` |
| `specs.js` | specs | `/api/v2/:db/specs/` — декларативные спецификации (инварианты) |
| `resolution.js` | resolution | `/api/v2/:db/resolution/` — golden record, MDM |
| `pm.js` | pm | `/api/v2/:db/pm/` — `listIssues`, `getIssue`, `createIssue`, `updateIssue`, `deleteIssue`; `listSprints`, `createSprint`, `updateSprint`; `getBoard`, `getBacklog`; `getMetrics(db, type)` (velocity, burndown, cycleTime, workload) |
| `ws.js` | — | WebSocket клиент |

## ws.js (WebSocket)

Подключается к `/ws` с JWT-токеном. Используется для:
- Push-уведомлений
- Коллаборативного редактирования документов
- Обновлений реального времени

Экспортирует `class WsClient` — WebSocket клиент с автоподключением, re-subscribe при разрыве, JWT-refresh перед реконнектом. Composable `useMultiWs` живёт в `composables/useMultiWs.js` и создаёт экземпляр `WsClient`.
