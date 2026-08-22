# Stores (Pinia)

**Директория:** `src/stores/`

| Файл | Store ID | Назначение |
|------|----------|-----------|
| `auth.js` | `auth` | Глобальный JWT, пользователь, логин/логаут/refresh |
| `workspace.js` | `workspace` | Список воркспейсов, настройки, включённые модули |
| `ui.js` | `ui` | Хлебные крошки навигации, заголовок страницы |
| `notifications.js` | `notifications` | Непрочитанные уведомления, polling |
| `aiChat.js` | `aiChat` | Conversation ID, сообщения AI-чата, SSE стрим |
| `normalizer.js` | `normalizer` | Состояние нормализации документов |
| `calls.js` | `calls` | Активный звонок, участники, локальные треки |
| `teamchat.js` | `teamchat` | Комнаты, топики, сообщения, участники, задачи |
| `decisions.js` | `decisions` | Решения, ссылки, обсуждения |
| `pm.js` | `pm` | Управление проектами: задачи, спринты, доска, метрики |

## auth

**Ключевые поля:**
- `globalToken` — access token (в памяти, не в localStorage)
- `globalUser` — объект пользователя
- `currentDb` — текущий db identifier

**Ключевые методы:**
- `iamLogin(email, password)` — POST `/api/v2/iam/login`, сохраняет `globalToken`
- `iamMe()` — GET `/api/v2/iam/me`, восстанавливает пользователя при reload
- `iamRefresh()` — POST `/api/v2/iam/refresh` (cookie → новый access token)
- `iamLogout()` — POST `/api/v2/iam/logout`, очищает token и user
- `iamRegister(data)` — регистрация нового пользователя
- `iamUpdateMe(data)` — обновление профиля текущего пользователя
- `iamChangePassword(currentPassword, newPassword)` — смена пароля
- `resetPassword(email)` — запрос сброса пароля
- `confirmReset(token, newPassword)` — подтверждение сброса пароля

## workspace

**Ключевые поля:**
- `workspaceList` — массив всех воркспейсов пользователя
- `settings` — настройки текущего воркспейса (включённые модули)
- `db` — текущий db identifier
- `types` — список типов (схема) текущего воркспейса
- `documents` — список документов
- `reports` — список отчётов
- `v2Grants` — гранты доступа
- `loading` — флаг загрузки текущего воркспейса
- `typesTotal` — общее количество типов (включая за пределами текущей страницы)
- `typeDetails` — кеш деталей типов `{ [typeId]: typeData }` (заполняется ленивой загрузкой)
- `parentInfo` — `{ id, value, typeId }` — хлебные крошки ObjectList для дочерних таблиц
- `workspacesLoading` — загружается ли список воркспейсов
- `workspacesLoaded` — загружен ли список хотя бы раз
- `workspacesError` — флаг ошибки загрузки списка
- `credentialsTypeId` — computed: ID типа «Credentials» из схемы (для хранения секретов коннекторов)

**Ключевые методы:**
- `bootstrap(dbName)` — загружает все данные воркспейса за один запрос (схема, документы, отчёты, гранты)
- `loadWorkspaces({ force })` — загружает список воркспейсов (кешируется)
- `syncSettings(slug)` — синхронизирует `settings` из `workspaceList` для указанного slug/db
- `moduleEnabled(name)` — проверяет, включён ли модуль в текущем воркспейсе. По умолчанию `true` если не задан явно, кроме opt-in модулей (`calls`) — для них дефолт `false`
- `loadSchema(dbName)` — отдельная загрузка схемы (для WS-инвалидации)
- `loadDocuments(dbName)` — отдельная загрузка документов
- `loadReports(dbName)` — отдельная загрузка отчётов
- `loadV2Grants(dbName)` — отдельная загрузка грантов доступа
- `loadType(typeId, dbOverride)` — ленивая загрузка деталей типа с кешированием
- `invalidateType(typeId)` — инвалидация кеша типа
- `isAdmin(dbName)` — проверяет, является ли пользователь admin/owner воркспейса
- `isOwner(dbName)` — проверяет, является ли пользователь owner воркспейса
- `resetCache()` — сбрасывает кеш списка воркспейсов

## ui

**Ключевые поля:**
- `breadcrumbs` — текущий путь навигации (хлебные крошки)
- `pageTitle` — заголовок текущей страницы (также устанавливает `document.title`)

**Ключевые методы:**
- `setBreadcrumbs(items)` — установить хлебные крошки (массив `{ label, to? }`)
- `setPageTitle(title)` — установить заголовок; если пусто — показывает только APP_NAME

## notifications

**Ключевые поля:**
- `unreadCount` — количество непрочитанных
- `runningAutomations` — счётчик выполняющихся автоматизаций (для badge)
- `items` — список уведомлений
- `loading` — флаг загрузки списка
- `lastType` — тип последнего уведомления (для цвета badge)

**Ключевые методы:**
- `fetchCount(db)` — GET `/notifications/count`, обновляет `unreadCount`
- `fetchList(db, unreadOnly?)` — GET список уведомлений
- `markRead(db, id)` — отметить прочитанным (локально + API)
- `markAllRead(db)` — отметить все прочитанными
- `remove(db, id)` — удалить уведомление

`unreadCount` предзагружается в `bootstrap`. Real-time обновления через WebSocket: `notifications:new` → `unreadCount++` + toast, `automations:started` → `runningAutomations++`, `automations:finished` → `runningAutomations--`. Badge цвет: синий пульсирующий (автоматизация бежит) → зелёный (есть непрочитанные) → скрыт (всё прочитано). Polling каждые 15 сек как fallback.

## aiChat

**Ключевые поля:**
- `conversations` — список бесед текущего воркспейса
- `activeConversationId` — ID активной беседы (null если не выбрана)
- `messages` — история сообщений текущей беседы (`shallowRef` для производительности)
- `streaming` — идёт ли SSE-стрим
- `pendingHitl` — ожидающее HITL-подтверждение (null или объект с `callId`, `toolName`, `args`)
- `pendingElicit` — ожидающая elicitation-форма (null или объект с `schema`, `conversationId`)
- `streamingContent` — накапливаемый текст текущего ответа (побатечное обновление ~30fps)
- `streamingToolCalls` — инструменты текущего ответа в процессе стриминга (`shallowRef`)

**Ключевые методы:**
- `loadConversations(db)` — GET `/ai/conversations`
- `loadMessages(db, convId)` — GET `/ai/conversations/:id/messages`
- `createConversation(db, title?)` — POST `/ai/conversations`
- `sendMessage(db, text, convId, pageContext?, options?)` — отправить сообщение через SSE (`/ai/agent-chat`)
- `resumeHitl(db, approved)` — подтвердить/отклонить HITL (`/ai/agent-resume`); convId и callId разрешаются внутри из `pendingHitl`
- `answerElicit(db, answer)` — заполнить elicitation-форму (`/ai/agent-elicit`)
- `cancelStream()` — прервать текущий SSE-стрим
- `deleteConversation(db, convId)` — DELETE беседы
- `clearMessages()` — очистить историю (вызывается при смене воркспейса)
- `sendFeedback(db, conversationId, messageIndex, signal, agentId)` — POST `/ai/feedback` (thumbs up/down)

## normalizer

Мультизадачный стор — отслеживает несколько заданий одновременно с периодическим polling'ом (6 сек).

**Ключевые поля:**
- `jobs` — массив заданий `[{ jobId, db, status: { stage, progress, errors }, meta }]`
- `activeJobs` — computed: задания в активных стадиях (completed/error/cancelled отфильтрованы)
- `hasActive` — computed boolean: есть ли активные задания

**Стадии (`status.stage`):**
`queued` → `listing` → `classifying` → `extracting` → `resolving` → `architecting` → `populating` → `writing` → `completed` | `error` | `cancelled`

**Ключевые методы:**
- `addJob(db, jobId)` — добавить задание и запустить polling
- `removeJob(jobId)` — убрать задание из списка
- `cancelJob(db, jobId)` — отменить задание через API
- `stopAll()` — остановить polling (для размонтирования)
- `stageLabel(stage)` — локализованная строка стадии
- `stageProgress(stage)` — прогресс 0–100 для прогресс-бара

## teamchat (`useTeamchatStore`)

**Ключевые поля:**
- `rooms` — массив комнат чата
- `currentRoom` — текущая выбранная комната
- `topics` — массив топиков текущей комнаты
- `messages` — массив сообщений текущего топика
- `members` — участники комнаты
- `loading` — флаг загрузки
- `wsConnected` — флаг подключения WebSocket
- `recentTopics` — массив недавних топиков (cross-room)
- `mutedTopicIds` — `Set` замьюченных топиков (клиентская фильтрация WS-сообщений)
- `tasks` — массив задач (топики с `assigned_to`)
- `tasksLoading` — флаг загрузки задач
- `tasksNextCursor` — курсор пагинации задач

**Computed:**
- `currentRoomId` — ID текущей комнаты (`currentRoom?.id`)
- `roomById` — `{ [id]: room }` lookup

**Ключевые методы:**

*Комнаты:*
- `loadRooms(db)` — GET `/teamchat/rooms`
- `createRoom(db, data)` — POST `/teamchat/rooms`
- `joinRoom(db, roomId)` — GET комнату и установить как текущую (+ WS join)
- `leaveRoom()` — сбросить текущую комнату, топики, сообщения, участников
- `loadRoom(db, roomId)` — GET `/teamchat/rooms/:roomId` (обновляет в массиве)
- `updateRoom(db, roomId, data)` — PATCH `/teamchat/rooms/:roomId`
- `deleteRoom(db, roomId)` — DELETE `/teamchat/rooms/:roomId`
- `joinPublicRoom(db, roomId)` — вступить в публичную комнату
- `loadPublicRooms(db)` — GET список публичных комнат

*Топики:*
- `loadTopics(db, roomId)` — GET `/teamchat/rooms/:roomId/topics`
- `createTopic(db, roomId, data)` — POST `/teamchat/rooms/:roomId/topics`
- `deleteTopic(db, topicId)` — DELETE топика + очистка сообщений
- `loadRecentTopics(db, params)` — GET недавних топиков (cross-room)

*Сообщения:*
- `loadMessages(db, topicId, params?)` — GET `/teamchat/topics/:topicId/messages` (cursor pagination, replaces messages)
- `loadOlderMessages(db, topicId, params?)` — GET `/teamchat/topics/:topicId/messages` (prepends older messages, returns them for scroll-to-load-more)
- `sendMessage(db, topicId, data)` — POST `/teamchat/topics/:topicId/messages`
- `deleteMessage(db, msgId)` — DELETE `/teamchat/messages/:msgId`
- `moveMessage(db, msgId, targetTopicId, sendNotices?, reloadTopicId?)` — переместить сообщение в другой топик
- `moveMessagesBulk(db, msgIds, targetTopicId, sendNotices?, reloadTopicId?)` — bulk-перемещение

*Участники:*
- `loadMembers(db, roomId)` — GET `/teamchat/rooms/:roomId/members`

*Muting и прочитанность:*
- `loadMutedTopics(db)` — загрузить список замьюченных топиков
- `muteTopic(db, topicId)` / `unmuteTopic(db, topicId)` — замьютить/размьютить
- `markTopicRead(db, topicId)` — пометить как прочитанный
- `isTopicMuted(topicId)` — проверить мьют

*Summarization и экспорт:*
- `summarizeTopic(db, topicId, since)` — AI-суммаризация топика
- `exportTopicToDocument(db, topicId, opts)` — экспорт топика в документ
- `refreshDocument(db, topicId)` — обновить привязанный документ

*Deep links:*
- `getTopicLink(db, topicId, params)` — получить ссылку на топик
- `getMessageLink(db, msgId)` — получить ссылку на сообщение

*Задачи:*
- `loadTasks(db, params?)` — GET задачи с cursor-пагинацией

*Прочее:*
- `setWsConnected(v)` — установить статус WS
- `reset()` — сбросить всё состояние

## decisions (`useDecisionsStore`)

**Ключевые поля:**
- `decisions` — массив решений
- `total` — общее количество решений
- `currentDecision` — текущее выбранное решение
- `links` — ссылки текущего решения
- `discussions` — обсуждения, связанные с решением
- `conflicts` — конфликты текущего решения
- `history` — история изменений текущего решения
- `loading` — флаг загрузки

**Computed:**
- `decisionById` — `{ [id]: decision }` lookup по ID

**Ключевые методы:**
- `loadDecisions(db, params?)` — GET `/decisions`
- `createDecision(db, data)` — POST `/decisions`
- `loadDecision(db, id)` — GET `/decisions/:id`
- `updateDecision(db, id, data)` — PATCH `/decisions/:id`
- `deleteDecision(db, id)` — DELETE `/decisions/:id`
- `loadLinks(db, decisionId)` — GET `/decisions/:id/links`
- `createLink(db, decisionId, data)` — POST `/decisions/:id/links`
- `deleteLink(db, decisionId, linkId)` — DELETE `/decisions/:id/links/:linkId`
- `loadDiscussions(db, decisionId)` — GET `/decisions/:id/discussions`
- `loadConflicts(db, decisionId)` — GET `/decisions/:id/conflicts`
- `loadHistory(db, decisionId)` — GET `/decisions/:id/history`
- `reset()` — сбросить всё состояние стора

## calls (`useCallsStore`)

WebRTC P2P — состояние входящего/текущего звонка и пиров.

**State:**
- `incoming` — объект входящего звонка (`{ callId, fromUserId, fromUserName, accept, reject }`) или `null`
- `current` — текущий звонок: `{ kind: 'direct'|'room', callId, ... }` или `null`
- `peers` — `Map<userId, { stream, audioLevel, ... }>` — состояние удалённых участников
- `localStream` — `MediaStream` локального микрофона/камеры (`null` если не получен)
- `mutedAudio` / `mutedVideo` — флаги mute локальных треков
- `screenSharing` — true если идёт демонстрация экрана
- `roomPresence` — массив `{ userId, name }` присутствующих в воркспейс-комнате

**Computed:**
- `roomCount` — `roomPresence.length`
- `inCall` — true если `current !== null`

**Actions:**
- `setIncoming(call)` / `clearIncoming()`
- `setCurrent(call)` / `clearCurrent()`
- `upsertPeer(userId, patch)` / `removePeer(userId)`
- `setRoomPresence(users)`

## pm (`usePmStore`)

Управление проектами — задачи, спринты, доска.

**State:**
- `issues` — массив задач
- `sprints` — массив спринтов
- `board` — `{ columns: {} }` — Kanban-доска
- `backlog` — задачи без спринта
- `activeIssue` — выбранная задача
- `loading` — флаг загрузки

**Computed:**
- `activeSprint` — текущий активный спринт
- `planningSprints` — спринты в планировании
- `issueById` — `Map<id, issue>`

**Actions:**
- `loadIssues(db, filters)` — GET `/pm/issues`
- `loadBoard(db, params)` — GET `/pm/board`
- `loadSprints(db)` — GET `/pm/sprints`
- `loadBacklog(db)` — GET `/pm/backlog`
