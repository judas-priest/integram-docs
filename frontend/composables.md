# Composables

**Директория:** `src/composables/`

Перечень сверен с каталогом перебором (`ls *.js` минус `*.test.js` против имён в
таблице). Добавляя composable — добавляй строку сюда: неполный перечень хуже
пустого, пустой виден, неполный выглядит полным.

| Файл | Назначение |
|------|-----------|
| `useApiCall.js` | Обёртка для API-вызовов: loading, error, автоматический toast при ошибке |
| `useAppToast.js` | Обёртка над PrimeVue Toast: `success()`, `error()`, `warn()`, `info()` |
| `useListLoader.js` | Типовой паттерн загрузки списка: `{ items, loading, loadData } = useListLoader(fetchFn, options?)`. Нормализация ответа, loading-флаг, toast при ошибке. `options.normalize` (default true) — `Array.isArray ? res : res.data ?? []`. `options.transform` — пост-обработка. |
| `useWidgetData.js` | Dashboard widget boilerplate: `inject` для dashFilter/crossFilter/refreshKey/setCrossFilter, `buildFilterParams(params, typeId)` — добавляет date range + crossFilter в params, `useWidgetLifecycle(loadData)` — wires watch + onMounted. Используется в KpiWidget, GalleryWidget, KanbanWidget, ChartWidget, TableWidget, CardWidget. |
| `useConfirmAction.js` | PrimeVue ConfirmDialog для подтверждения деструктивных операций |
| `useInfiniteScroll.js` | Infinite scroll с IntersectionObserver |
| `useMultiWs.js` | Менеджер WebSocket-соединений: подключение к нескольким воркспейсам |
| `usePresence.js` | Shared presence handling: `collaborators` Map + `handlePresenceMessage(msg)` для `presence-join/leave/list`. Опции: `extraFields(userId)` — доп. поля в запись коллаборатора, `onLeave(userId)` — коллбэк при уходе. Используется в `useTableSync` и `useDocumentSync`. |
| `useTableSync.js` | Real-time таблица через WS: presence (через `usePresence`), cell-focus, remote CRUD. Подписка на `objects` канал, relay `change` events в callback. |
| `useTypeLoader.js` | Ленивая загрузка схемы типа по typeId с кешированием |
| `useSlashCommands.js` | Slash-команды в блочном редакторе (`/heading`, `/code`, ...) |
| `useMentionBase.js` | Базовая логика popup-упоминаний: `isOpen`, `query`, `selectedIndex`, `position`, `open`, `close`, `onKeydown(e, filteredItems, insertFn)`, `onTextChange(quill, updateQueryFn)`, `insertAtCursor(blotType, data)`. Используется в `useMentions` и `useRecordMentions`. |
| `useMentions.js` | @-упоминания пользователей в редакторе. Использует `useMentionBase`. |
| `useRecordMentions.js` | @-упоминания записей (объектов) в редакторе. Использует `useMentionBase`. |
| `useSmartPaste.js` | Умная вставка: определяет тип вставляемого контента |
| `useUndoRedo.js` | История изменений: undo/redo для редакторов |
| `useKeyboardShortcuts.js` | Регистрация и управление горячими клавишами |
| `useSavedViews.js` | Управление сохранёнными видами таблицы. API: `views`, `activeViewId`, `isModified` (dirty tracking), `loadViews`, `applyView`, `clearActiveView`, `trackCurrentConfig`, `serializeTableState` (20+ полей: density, aggConfig, kanbanCardFields, colorByField, timeline/gantt), `saveCurrentAsView`, `updateView`, `renameView`, `toggleShareView`, `deleteView` |
| `useBlockControls.js` | Контролы блока в блочном редакторе (drag, меню) |
| `useBlockEditor.js` | Инициализация и управление экземпляром блочного редактора |
| `useEditorOps.js` | Операции редактирования: вставка, удаление, трансформация блоков |
| `useDocumentSync.js` | Синхронизация документа через WebSocket (коллаборация) |
| `useWorkspaceWs.js` | Shared WS connection per workspace. `on()` listeners survive reconnect (watches persist across disconnect/connect cycles). `subscribe()` consumers re-registered on auth OK. |
| `useWorkspaceSync.js` | Sidebar real-time sync. Channels: `schema`, `documents`, `workspaces`, `notifications` (toast + badge), `automations` (started/finished toast + runningAutomations counter). |
| `useAIChat.js` | SSE-based AI chat composable. Two modes: `agentSlug=null` (conversations) or `agentSlug='xxx'` (topic-based). Returns reactive state (messages, streaming, pendingHitl, pendingElicit, streamingContent, streamingToolCalls, conversations, conversationsLoading, activeConversationId, activeTopicId, debateState, messagesLoading, currentTraceId) and methods (sendMessage, loadConversations, loadMessages, loadTopicMessages, resumeHitl, answerElicit, cancelStream, clearMessages). `sendMessage(db, text, convId, pageContext?, options?)` accepts `searchMode` option ('precise'\|'balanced'\|'full') for Meta KB precision control and `fileIds` array for attaching uploaded files. `loadTopicMessages(db, topicId)` loads topic messages + saved debate state asynchronously. Used by `stores/aiChat.js` (thin wrapper) and Meta KB view. |
| `useSSEStream.js` | Управление SSE-потоком для AI-стриминга |
| `useModule.js` | Проверка доступности модуля в текущем воркспейсе |
| ~~`usePageTitle.js`~~ | Удалён — функционал перенесён в `useUiStore` (`stores/ui.js`, `setPageTitle`) |
| `usePortalEditor.js` | Состояние и операции визуального редактора портала. Включает undo/redo через `useUndoRedo` — история изменений конфига с Ctrl+Z/Ctrl+Y. |
| `useTimelineBase.js` | Shared zoom / date / header logic for GanttView and TimelineView. Exports: composable `useTimelineBase({ items, scrollElRef, minTotalWidth })` + standalone `findNameCol(columns)`, `parseTimelineDate(raw)`, constants `ZOOM_LEVELS`, `ZOOM_DAY_WIDTH`, `MONTH_NAMES`. |
| `useChildRecords.js` | Загрузка и управление child-таблицами родительской записи. `useChildRecords({ typeInfo, db, objectId, workspace })` → `{ childTypes, childRecords, childBoolCols, loadChildRecords, getChildBoolValue, toggleChildBool }`. Bool-колонки (type=11) автоматически отделяются для inline-чекбоксов с оптимистичным обновлением. Используется в ObjectView и RecordDrawer. |
| `useResolvedCellValue.js` | Резолвинг ref-колонок (числовых ID) в формат `"id:label"` для CellRenderer. `useResolvedCellValue({ db })` → `{ lookupOptions, loadLookup, cellCol, resolvedCellValue }`. `cellCol(col)` добавляет `isReference: true` для ref-колонок. `resolvedCellValue(raw, col)` конвертирует `"18"` → `"18:John Doe"`. Используется в ObjectView и RecordDrawer. |
| `useCellSelection.js` | Excel-like мультиселекция ячеек в DataTable. Drag для прямоугольника, Ctrl+click toggle, Shift+click extend, triple-click = строка, Shift+Arrow расширение. Floating badge Σ/N/⌀. TSV copy (Ctrl+C) для вставки в Excel/Google Sheets. |
| `useCall.js` | WebRTC P2P звонки: 1-1 + workspace voice room (≤8). См. секцию ниже. |
| `usePointPresence.js` | Клиент WS-канала `point` — «кто сейчас смотрит эту запись». `{ viewers, join, leave }`. См. секцию ниже. |
| `useYjsSync.js` | Yjs-провайдер поверх общего WS воркспейса, канал `sandbox-collab`. Y.Doc на ячейку кода, awareness (курсоры/присутствие), периодическое автосохранение состояния через `sandboxService`. `useYjsSync({ db, topicId, messageId, initialContent, yjsState })`. |
| `useComputedColumns.js` | Вычисляемые реквизиты типа (LOOKUP / ROLLUP / FORMULA) как виртуальные колонки. `useComputedColumns({ db, typeId })` → `{ computedReqs, computedColumns, loadComputedReqs }`. Ключ значения совпадает с id колонки (`computed_<defId>`) и в списке, и в одиночной записи. Поколение загрузки защищает от перезаписи свежего ответа устаревшим. |
| `useReportLookups.js` | Подгрузка справочных значений для ref-колонок отчёта. `useReportLookups(db)` → `{ lookupMap, loading, loadLookups, getOptions, isRef }`. `lookupMap` — `{ [reqTypeId]: [{ id, label }] }`. |
| `useTts.js` | Озвучивание ответов. Настройки в `localStorage` (`tts_settings`): `enabled`, `autoPlay`, `engine` (`browser` \| `piper`), `voiceId`, `speed` и `scope` по разделам (aiChat, metaKb, teamChat, debate). Текст чистится `stripForTts`. |
| `useSchemaOptions.js` | Загрузка метаданных схемы (таблицы, колонки, записи) для каскадных выпадающих списков в формах. См. секцию ниже. |
| `usePmShortcuts.js` | Горячие клавиши раздела PM: `c` (и русская `с`) — создать задачу, `1`…`6` — переключить вкладку. Не срабатывает в полях ввода и `contenteditable`. `usePmShortcuts({ onCreateIssue, onSwitchTab })`. |

## usePointPresence()

Клиент WS-канала `point`. Возвращает `{ viewers, join, leave }`, где `viewers` —
реактивная `Map` `userId → { username, color }` **без себя самого**.

```js
const { viewers, join, leave } = usePointPresence();
join('record', objectId);   // повторный вызов сначала уводит с прежней точки
```

Две вещи, из-за которых голого `ws.subscribe` не хватает:

1. **`WsClient.unsubscribe()` серверу не пишет** (`services/ws.js`) — он только
   чистит локальную карту. Отписка отправляется кадром вручную в `leave()`, иначе
   сервер не разошлёт `presence-leave` и не запишет уход в журнал присутствия.
2. **Приходящее по каналу надо фильтровать по своей точке** — обработчик `on()`
   получает ВСЕ кадры типа `point`, включая чужие точки.

`leave` навешан на `onBeforeUnmount`. `pointType` берётся из словаря платформы
(`table` / `record`), а не выдумывается. Серверная половина —
`backend/docs/modules/presence.md`.

## useCall(`{ db, myUserId }`)

Управляет mesh-сетью `RTCPeerConnection`, WebSocket-сигналлингом по каналу `calls`, screen-share и ICE-restart таймерами.

**Public methods:**
- `invite(targetUserId)` — позвать пользователя (1-1)
- `accept(call)` / `reject(call)` — обработать входящий
- `hangup()` — завершить текущий звонок
- `joinRoom({ withVideo, withAudio })` / `leaveRoom()` — войти/выйти в workspace voice room
- `toggleMic()` / `toggleCam()` / `toggleScreenShare()` — управление локальными треками

**Внутренние особенности:**
- Per-peer Perfect Negotiation: `polite = myUserId < remoteUserId`
- ICE-restart таймер 5 секунд при переходе PC в `disconnected`
- Polling уровня аудио (1×/с) для UI индикатора говорящего
- TURN credentials загружаются лениво при создании первого PC (`callsService.turnCredentials(db)`)
- WS-подписка на канал `calls` через `useWorkspaceWs().on()`; отписка в `onBeforeUnmount`

### useSchemaOptions

Composable for loading schema metadata (tables, columns, records) for form pickers.

```js
import { useSchemaOptions } from '@/composables/useSchemaOptions';

const schema = useSchemaOptions();
await schema.loadTables();

// Table picker
schema.tableOptions.value // [{label, value}]

// Column picker (cascading from table)
await schema.ensureColumns(typeId);
schema.columnsFor(typeId) // [{label, value, type}]

// Record picker (for goFilter values)
await schema.recordsFor(typeId) // [{label, value}]

// Find ref target table for a column
// Returns null for non-ref columns AND for system types (id < 100)
schema.refTargetTypeId(tableTypeId, columnReqId) // number | null
```

Used by `ActionCard.vue` to replace manual numeric ID inputs with cascading dropdowns.
