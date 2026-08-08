# Views — Страницы

**Директория:** `src/views/`

Один файл = одна страница. Каждая view связана с маршрутом в `router/index.js`.

## Корневые

| Файл | Маршрут | Назначение |
|------|---------|-----------|
| `Landing.vue` | `/` | Лендинг для неавторизованных пользователей |
| `Dashboard.vue` | `/:db/` | Главная страница воркспейса (workspace home) |

## auth/

| Файл | Маршрут | Назначение |
|------|---------|-----------|
| `Login.vue` | `/login` | Форма входа. Workspace picker: секция "Недавние" (localStorage, max 5, не дублируется в основном списке), сортировка (новые/старые/А-Я, цикличная кнопка, хранится в localStorage). При >5 воркспейсов поиск получает autofocus. |
| `Register.vue` | `/register` | Регистрация |
| `ForgotPassword.vue` | `/forgot-password` | Запрос сброса пароля |
| `ResetConfirm.vue` | `/reset-confirm` | Подтверждение нового пароля |
| `OtpVerify.vue` | `/otp` | Верификация OTP |
| `AutoLogin.vue` | `/auto-login/:token` | Автологин по токену |
| `AccessDenied.vue` | `/access-denied` | 403 страница |
| `Error.vue` | `/error` | Общая ошибка |
| `NotFound.vue` | `*` | 404 страница |
| `DatabaseSelector.vue` | — | Выбор воркспейса после логина |

## data/

| Файл | Маршрут | Назначение |
|------|---------|-----------|
| `DataHome.vue` | `/:db/tables` | Список всех таблиц |
| `ObjectList.vue` | `/:db/tables/:typeId` | Список записей таблицы (все виды: grid, kanban, calendar, ...). Multi-row Excel paste создаёт новые записи через `onPasteRows()` handler. |
| `ObjectView.vue` | `/:db/tables/:typeId/:id` | Просмотр записи. `_value`/isPrimary-колонка читается из `obj.value.value` (не из requisites); ref-значения резолвятся через lookupOptions в формат `id:label` для CellRenderer |
| `ObjectEdit.vue` | `/:db/tables/:typeId/:id/edit` | Редактирование записи. `_value` (primary column) отправляется как `body.value`, не внутри `requisites` — иначе backend вернёт "Unknown column" |
| `ObjectImport.vue` | `/:db/tables/:typeId/import` | CSV импорт |
| `TrashView.vue` | `/:db/trash` | Корзина |
| `PublicRecord.vue` | `/public/records/:token` | Публичная запись |
| `PublicTableView.vue` | `/public/v/:token` | Публичный вид таблицы |

## schema/

| Файл | Маршрут | Назначение |
|------|---------|-----------|
| `TypeList.vue` | `/:db/schema` | Список типов (таблиц). LOD: `showColumns = zoomLevel >= 0.4` — ниже порога показывает "Приблизьте для просмотра колонок" |
| `TypeCreate.vue` | `/:db/schema/new` | Создание типа |
| `TypeEdit.vue` | `/:db/schema/:typeId` | Редактирование типа, колонок, computed |
| `components/ColumnsEditor.vue` | — | Редактор колонок (вложен в TypeEdit). Использует `COLUMN_TYPES_FULL`/`TYPE_LABEL_MAP` из `@/constants/columnTypes.js` как единый источник типов; ref-колонки имеют `col.type=1` (legacy), реальная цель — `col.refTypeId` |
| `components/ComputedReqsEditor.vue` | — | Редактор computed columns |
| `components/TypeNameEditor.vue` | — | Редактор имени типа (вложен в TypeEdit) |

## reports/

| Файл | Маршрут | Назначение |
|------|---------|-----------|
| `ReportList.vue` | `/:db/reports` | Список отчётов |
| `ReportView.vue` | `/:db/reports/:id` | Просмотр отчёта. Подписывается на WS-события через `useTableSync` по `parentTypeId` (из структуры отчёта) — автоматически перезагружает данные при изменении строк без перезагрузки страницы. Удаления применяются мгновенно (без full reload). **Переименование колонки**: V-стрелка / правый клик → Popover «Переименовать» → `tableRef.startHeaderRename()` → инлайн-ввод в заголовке, Enter сохраняет через `reportsService.renameColumn`. **Группировка**: dot-naming в `displayName` (`"руб/ед.Комус"`) рендерит двухуровневый header через DataTable smart-header. **SSE streaming**: toggle для больших отчётов — загрузка данных через SSE (`reportsService.stream`) вместо одного GET. **Bulk-update**: диалог массового обновления с превью изменений и подтверждением. |
| `ReportEdit.vue` | `/:db/reports/:id/edit` | Редактор отчёта |

## dashboards/

| Файл | Маршрут | Назначение |
|------|---------|-----------|
| `DashboardList.vue` | `/:db/dashboards` | Список дашбордов |
| `DashboardBuilder.vue` | `/:db/dashboards/:id` | Редактор дашборда с виджетами. Типы: chart, kpi, table, text, kanban, gallery, pivot, report, document, card. CrossFilter поддерживает `parentId` для кросс-табличных связей (клик по дочерней записи → шапка родителя). |

## documents/

| Файл | Маршрут | Назначение |
|------|---------|-----------|
| `DocumentList.vue` | `/:db/documents` | Список документов |
| `DocumentEditor.vue` | `/:db/documents/:docId` | Блочный редактор (Quill) |
| `DocumentVersions.vue` | `/:db/documents/:docId/versions` | История версий |
| `DocumentSharing.vue` | `/:db/documents/:docId/sharing` | Управление доступом |
| `PublicDocument.vue` | `/public/docs/:db/:id` | Публичный документ |

## automations/

| Файл | Маршрут | Назначение |
|------|---------|-----------|
| `AutomationsDashboard.vue` | `/:db/automations` | Сводный дашборд автоматизаций |
| `AutomationList.vue` | `/:db/automations/list` | Список автоматизаций — бейдж "Системная" для `isSystem=true` |
| `AutomationEdit.vue` | `/:db/automations/:id` | Редактор автоматизации. Переменные шаблона: `{{id}}`, `{{val}}`, `{{typeId}}`, `{{created_by}}`, `{{_screen}}`, `{{_page}}`, `{{_items}}` + `{{req_XXX}}` для полей таблицы-триггера. **Batch run**: кнопка массового запуска с polling прогресса (`automationsService.runBatch` / `batchStatus`). |
| `AutomationRuns.vue` | `/:db/automations/:id/runs` | Лог запусков |

## portal/

| Файл | Маршрут | Назначение |
|------|---------|-----------|
| `PortalVisualEditor.vue` | `/:db/portal-editor` | WYSIWYG-редактор портала |
| `PortalConfigPage.vue` | — | Устаревший конфиг (заменён PortalVisualEditor) |

## ai/

| Файл | Маршрут | Назначение |
|------|---------|-----------|
| `AiChat.vue` | `/:db/ai` | AI-чат. Два режима: AI Ассистент (conversations) и Meta KB (teamchat-agent, topic-based). History popover со списком бесед/топиков. В Meta KB: search precision selector (precise/balanced/full), панель дебатов экспертов (DebatePanel), AgentSteps timeline, CitationRenderer с inline-цитатами, follow-up chips (предлагаемые вопросы). Welcome screen с примерами запросов для каждого режима. |
| `AiFillTable.vue` | `/:db/ai/fill/:typeId` | AI-заполнение колонки |
| `AiColumnConfig.vue` | `/:db/ai/column-config/:typeId/:reqId` | Настройка AI-кнопки |

## admin/

| Файл | Маршрут | Назначение |
|------|---------|-----------|
| `AdminDashboard.vue` | `/:db/admin` | Сводная панель администратора. Секция обслуживания: кнопка пересоздания flat views (`schemaService.rebuildFlatViews`). |
| `V2Grants.vue` | `/:db/admin/v2-grants` | Гранты доступа |
| `Roles.vue` | `/:db/admin/roles` | Роли |
| `RowRules.vue` | `/:db/admin/row-rules` | Row-level правила |
| `Backup.vue` | `/:db/admin/backup` | Резервное копирование |
| `Export.vue` | `/:db/admin/export` | Экспорт данных |
| `DeadLetterQueue.vue` | `/:db/admin/dlq` | Dead Letter Queue |
| `AuditLog.vue` | `/:db/admin/audit` | Журнал аудита |
| `QaBugs.vue` | `/:db/admin/qa-bugs` | QA баги |
| `QaResults.vue` | `/:db/admin/qa-results` | QA результаты |
| `TgBotList.vue` | `/:db/admin/telegram-bots` | List of workspace Telegram bots with CRUD |
| `TgBotEdit.vue` | `/:db/admin/telegram-bots/:id` | Bot editor — config, commands, keywords, sync, status, test |
| `Users.vue` | — | Управление пользователями (не привязан к маршруту) |

## Остальные группы

| Группа | Views | Назначение |
|--------|-------|-----------|
| `files/` | `FileManager.vue` | Файловый менеджер |
| `forms/` | `FormList`, `FormBuilder`, `FormPublic` | Публичные формы |
| `webhooks/` | `WebhookList`, `WebhookEdit` | Вебхуки. `WebhookEdit`: сворачиваемая секция истории доставок с кнопкой retry (`webhooksService.deliveries` / `retryDelivery`). |
| `connectors/` | `ConnectorList`, `ConnectorEdit`, `ConnectorRun` | Коннекторы |
| `graph/` | `GraphExplorer`, `GraphPathFinder` | Граф знаний. `GraphExplorer`: toggle панели KAG-поиска (`KagSearchPanel`). |
| `timeseries/` | `TimeseriesView` | Метрики |
| `codespace/` | `CodespaceList`, `RepoView` | Git-репозитории. `RepoView` поддерживает: создание/удаление веток (диалог + dropdown), создание файла (диалог с путём и содержимым), удаление файла (кнопка в FileTree), коммит нескольких файлов атомарно, переоткрытие PR + редактирование заголовка/описания прямо в PrDrawer, кнопка GitHub Sync в тулбаре (открывает `GithubSyncPanel`). |
| `normalizer/` | `NormalizerWizard`, `NormalizerReview` | AI-нормализатор |
| `workspaces/` | `WorkspaceList`, `WorkspaceCreate`, `WorkspaceEdit`, `WorkspaceMembers` | Воркспейсы — `WorkspaceCreate` содержит multi-step wizard (smart/manual mode), см. `wizard/` подпапку. `WorkspaceEdit` AI-вкладка содержит кнопку "Восстановить системные автоматизации" (`POST /automations/seed-system`) |
| `workspaces/wizard/` | `WizardStepMode`, `WizardStepDescribe`, `WizardStepPreview`, `WizardStepUpload`, `WizardStepManual`, `WizardStepConnectors` | Компоненты wizard создания воркспейса. `WizardStepDescribe` — описание бизнеса + вызов `/wizard/analyze`. `WizardStepPreview` — карточки сущностей + модули + рефайн (`/wizard/adapt`). `WizardStepUpload` — загрузка данных (CSV/Excel) + запуск нормалайзера. `WizardStepManual` — ручной режим с выбором шаблона. `WizardStepConnectors` — ввод API-токенов для коннекторов шаблона (вызывает `fillConnectorTokens` + запускает коннекторы). |
| `orgs/` | `OrgList`, `OrgEdit`, `OrgMembers` | Организации |
| `settings/` | `Profile`, `ChangePassword`, `ApiKeys` | Настройки пользователя |
| `notifications/` | `NotificationList` | Уведомления |
| `desktop/` | `DesktopView`, `DesktopTabContent`, `AddTabDialog` | Рабочий стол |
| `agents/` | `AgentList` | Реестр внешних агентов. Монтирует `SuggestionInlineBlock` — блок pending-предложений от behavioral analysis. |
| `testing/` | `TestingView` | Страницы тестирования |

## teamchat/

| Файл | Маршрут | Назначение |
|------|---------|-----------|
| `TeamChatLayout.vue` | `/:db/teamchat` | Основная страница чата: список комнат, топиков, сообщений. Поддерживает cursor-пагинацию, reply_to, mention-ссылки, scroll-to-load-more (prepend), deep-links `?topic=N` и `?msg=N`, claim-task. Кнопка справки (?) в заголовке списка комнат открывает HelpDialog с topic="teamchat". |
| `TaskList.vue` | (компонент внутри TeamChatLayout) | Список задач (топиков с assigned_to). Emits: `open-task`, `back`, `claim-task`. Inline-редактирование статуса, приоритета, исполнителя, дедлайна. |
| `TaskBoard.vue` | `/:db/teamchat/board/:roomId` | Канбан-доска задач по статусу/приоритету, данные через `teamchatService.getKanban` |

## calls/

| Файл | Маршрут | Назначение |
|------|---------|-----------|
| `ConferenceView.vue` | `/:db/conference` | Полноэкранная конференц-комната: вход/выход из workspace voice room |

## decisions/

| Файл | Маршрут | Назначение |
|------|---------|-----------|
| `DecisionList.vue` | `/:db/decisions` | Список решений с поиском и фильтрацией. Кнопка справки (?) в заголовке открывает HelpDialog с topic="decisions". |
| `DecisionDetail.vue` | `/:db/decisions/:id` | Просмотр/редактирование решения, его ссылок и обсуждений |
| `SearchWithAnamnesis.vue` | `/:db/decisions/search` | Полнотекстовый поиск по решениям с контекстом (анамнез) |
