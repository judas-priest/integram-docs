# Components

**Директория:** `src/components/`

## Общие компоненты

| Файл | Назначение |
|------|-----------|
| `EmptyState.vue` | Заглушка при пустом списке — иконка + текст + кнопка |
| `PageHeader.vue` | Заголовок страницы. Props: `title`. Slots: `#before` (перед заголовком, для стрелки назад), default (после заголовка, для кнопок действий). |
| `CardGridSkeleton.vue` | Skeleton-загрузчик для карточек |
| `ViewSwitcher.vue` | Переключатель видов таблицы (grid / kanban / calendar / ...) |
| `HistoryDrawer.vue` | Боковая панель истории изменений записи |
| `SchemaHistoryDrawer.vue` | Боковая панель истории изменений схемы |
| `SchemaSnapshotsDrawer.vue` | Drawer со списком снимков схемы (expand/collapse для JSON-деталей). Props: `db`, `typeId`. Используется в TypeEdit.vue. |
| `ReportHistoryDrawer.vue` | Drawer с аудит-логом отчёта (создание/изменение/удаление). Props: `db`, `reportId`. Используется в ReportEdit.vue. |
| `KagSearchPanel.vue` | Панель поиска сущностей KAG-графа знаний с traverse. Props: `db`. Встроена в GraphExplorer.vue. |
| `SearchDialog.vue` | Глобальный диалог поиска |
| `KeyboardShortcutsDialog.vue` | Диалог горячих клавиш |
| `HelpDialog.vue` | Диалог справки. Topics: `automations`, `webhooks`, `admin-grants`, `admin-row-rules`, `admin-backup`, `admin-export`, `admin-dlq`, `ai-chat`, `connectors`, `computed-fields`, `testing-auth`, `testing-workspaces`, `testing-schema`, `testing-records`, `testing-views`, `testing-relations`, `testing-documents`, `testing-reports`, `testing-dashboards`, `testing-automations`, `testing-ai`, `testing-forms`, `testing-admin`, `testing-files`, `testing-connectors`, `testing-portal`, `testing-graph`, `testing-notifications`, `testing-metrics`, `column-types`, `teamchat`, `decisions`. |
| `RequisiteField.vue` | Универсальное поле реквизита (по типу колонки). Поддерживает: текст (3,8), memo (12, Textarea), HTML (2, rendered preview + Textarea), markdown (1019, rendered preview + Textarea), число (13,14), дата (9), дата+время (4), bool (11), файл (10), пароль (6), uuid (1017), ref (single/multi), select/status (1006,1007), автономер/формула/AI-колонка (1009,15,1018, read-only), системные поля (1010-1014, read-only), URL (1001), валюта (1002), процент (1003, 0-100%), длительность (1004, секунды), рейтинг (1005, 1-5 звёзд), телефон (30, type=tel), email (41, type=email). AI-кнопка (1015): скрыта при создании, кнопка при редактировании (emits ai-run). Пропсы: objectId, aiRunning. |
| `AddColumnForm.vue` | Форма добавления новой колонки |
| `AiAutofillPanel.vue` | Панель AI-автозаполнения |
| `AiToolCallCard.vue` | Карточка вызова AI-инструмента. Prop `expanded` (bool) — развёрнутый вид с метками Meta KB инструментов и раскрываемым JSON-результатом |
| `AiPanel.vue` | Боковая AI-панель. Режимы: AI Ассистент (conversations) и Meta KB (teamchat-agent). Meta KB показывает карточки решений, конфликтов и источников через MessageCards. Переключатель виден admin/owner |
| `AiHitlDialog.vue` | Диалог HITL-подтверждения для AI |
| `NormalizerWidget.vue` | Виджет статуса нормализатора |
| `CommentThread.vue` | Ветка комментариев к записи |
| `EmojiPicker.vue` | Выбор эмодзи |
| `FluentEmoji.vue` | Отображение Fluent Emoji |
| `RecordPicker.vue` | Async-поиск и выбор EAV-записи из указанной таблицы. Пропсы: `typeId` (Number, required), `modelValue` (Number, ID выбранной записи), `placeholder` (String). Emits: `update:modelValue` (Number|null). Использует PrimeVue AutoComplete, ищет через `objectsService.list` с параметрами `typeId` и `search`. |
| `MemberDialog.vue` | Модалка с базовой инфой о члене воркспейса (аватар, имя, email, @username, бейдж роли) и кнопкой «Позвонить». Пропсы: `visible` (Boolean), `member` (Object — запись из `WorkspaceMembers`), `currentUserId` (Number). Emits: `update:visible`. Получает `call` через `inject('call')`, кнопка звонка скрыта для самого себя и при выключенном модуле `calls`. |
| `MembersPage.vue` | Переиспользуемая страница списка участников (воркспейса или организации). Props: `service` (Object — `{ getMembers, addMember, updateMember, removeMember }`), `slug` (String), `title` (String, default 'Участники'), `hideHeader` (Boolean), `rowClickable` (Boolean). Emits: `member-click`. Slots: `#header-buttons`, `#member-extra` (per member row), `#empty-actions`, `#after-list`, `#dialogs`. Expose: `openAdd()`, `members`, `loading`, `currentUserId`, `grantableRoleOptions`. Используется в `WorkspaceMembers.vue` и `OrgMembers.vue`. |

## calls/

WebRTC-компоненты P2P звонков. Все четыре монтируются из `AppLayout.vue` (gated by `workspace.moduleEnabled('calls')`).

| Файл | Назначение |
|------|-----------|
| `CallsContainer.vue` | Хост-компонент: создаёт `useCall({db, myUserId})` в своём setup, пишет результат в shared `callRef` (provide/inject из AppLayout). Рендерит `IncomingCallModal` + `CallWindow`. |
| `IncomingCallModal.vue` | PrimeVue Dialog: показывает имя звонящего + кнопки «Принять»/«Отклонить» (вызывают `props.call.accept`/`reject`). |
| `CallWindow.vue` | PrimeVue Dialog (80vw/80vh): grid `<video>` (локальный + по пиру) + контролы mic/cam/screen/hangup. |
| `RoomIndicator.vue` | Пункт сайдбара «Конференция» (workspace voice room). Использует класс `sidebar-nav-item`, иконка `pi-phone` (вне комнаты) / `pi-volume-up` (в комнате). Badge `roomCount` показывается когда в комнате есть участники (зелёный — ты внутри, синий — другие). Клик переключает join/leave. Рендерится в `AppMenu.vue`. |

## datatable/

Виды отображения записей таблицы и вспомогательные компоненты:

| Файл | Назначение |
|------|-----------|
| `DataTable.vue` | Основной компонент таблицы. Ключевые фичи: **double-click to edit** (Excel-поведение, одиночный клик — навигация/выбор); **row selection** — клик по пустому месту в row-counter ячейке выделяет строку, Ctrl+A выделяет все, Shift+клик — range select; при выделении появляются чекбоксы и bulk bar (удалить/изменить поле); grip скрывается в selection mode чтобы не перекрывать чекбокс; контекстное меню (grip/ПКМ) содержит пункт «Выделить строку»; **Ctrl+V paste** из TSV (Excel/Sheets → вставка в выделенный диапазон ячеек через `paste` event); **multi-row paste** — вставка нескольких строк из Excel автоматически создаёт новые записи (emits `paste-rows`); **cell styling** — focused cell = light blue background + blue border, editing cell = white background + blue border; **fill handle** — drag-handle на focused editable cell для протяжки значений (поддерживает _value, числа, даты, арифметические прогрессии); **smart header grouping** — dot-naming `"Группа.Колонка"` в `displayName` автоматически рендерит двухуровневый header (родительская строка + sub-row); **group separators** — CSS `:has(.smart-header-parent ~ .smart-header-parent)` добавляет `border-left` только когда групп ≥ 2; **expand dialog** — карточка редактирования записи, CellEditor для ref-полей монтируется с `autoFocus=false` (дропдаун не открывается автоматически); `defineExpose({ startHeaderRename })` — публичный метод для программного старта инлайн-переименования колонки. **Mobile (≤768px):** isNarrow ref (matchMedia) — отключает инлайн minWidth на table, колонки получают гибкие ширины (auto/80px/none), строки переходят в карточную вёрстку (display:block, data-label псевдо-элементы), thead скрыт, vs-spacer скрыт (виртуальный скролл отключен), панель настроек на всю ширину. |
| `ListView.vue` | Стандартный табличный список |
| `KanbanView.vue` | Канбан-доска. Props: `groupOptions` (lookup values for all columns), `groupCounts` (server-side counts per group), `cardLayout` (Object — конфигурация карточки: `{ title, subtitle, accent, secondary, badges: [] }` — слоты для отображения полей на карточке), `loading` (Boolean — включает skeleton-состояние загрузки), `hideEmpty` (Boolean), `canWrite` (Boolean). Emits: `row-click`, `card-drop`, `add-row`. Uses server-side grouped loading in ObjectList for large datasets (100 cards per column). |
| `GalleryView.vue` | Галерея карточек |
| `CalendarView.vue` | Календарный вид |
| `GanttView.vue` | Диаграмма Ганта |
| `TimelineView.vue` | Таймлайн |
| `PivotView.vue` | Сводная таблица |
| `CellEditor.vue` | Inline-редактор ячейки. Props: `autoFocus` (default `true`) — при `false` не фокусит input и не открывает ref-дропдаун при монтировании (используется в expand dialog). Ref/select/multi-ref дропдауны телепортируются в `<body>` с `v-if="dropdownOpen"` — рендерятся только при фокусе на input (`@focus`), закрываются при `blur`/`save`. |
| `CellRenderer.vue` | Отображение ячейки |
| `EmbeddedObjectTable.vue` | Встроенная таблица связанных объектов |
| `DependencyField.vue` | Поле зависимостей между объектами |
| `AiButtonConfigDialog.vue` | Диалог настройки AI-кнопки из таблицы. Поля: промпт, модель, outputReqId, temperature, maxTokens, systemPrompt, agentMode (переключатель — запускает полного агента вместо простого LLM). |
| `HttpButtonConfigDialog.vue` | Диалог настройки HTTP-кнопки (type 1016). Поля: метод, URL, заголовки, тело, responsePath, outputReqId. |
| `ScriptButtonConfigDialog.vue` | Диалог настройки Script Button (type 1020). Поля: JS-редактор (monospace textarea), список доступных полей строки, outputReqId. Кнопка «Запустить на строке 1» — auto-saves draft, затем выполняет в browser Web Worker, показывает результат инлайн. Глобалы в скрипте: `row`, `fetch`, `ai(prompt, model?)`, `output`, `setField(reqId, value)`, `createDocument(title, sections)`, `console` (log/warn/error), `JSON`, `Math`, `Date`. |
| `RecordDrawer.vue` | Slide-in боковая панель для просмотра и редактирования записи без перехода на отдельную страницу. Props: `db` (String, required), `typeId` (String\|Number). Emits: `update` (при сохранении изменений), `close` (при закрытии). Открывается программно через `expose`-метод `open(objectId)`. Отображает поля записи в двухколоночной сетке через CellRenderer, резолвит lookup-значения для ref-полей, показывает дочерние таблицы в табах. Кнопка «Открыть полностью» переходит на `/db/tables/typeId/objectId`. Кнопка-шестерёнка в шапке открывает Popover с MultiSelect для выбора и упорядочения отображаемых полей; выбор сохраняется в `localStorage` под ключом `drawer_prefs_${db}_${typeId}` (`visibleFieldIds: number[] \| null`, `null` = все поля). |
| `ImportDialog.vue` | Диалог импорта CSV |
| `TemplatesDialog.vue` | Диалог выбора шаблона |
| `FormulaEditorDialog.vue` | Редактор формул |

## block-editor/

Компоненты Quill-редактора блоков документа:

| Файл / Директория | Назначение |
|-------------------|-----------|
| `BlockEditor.vue` | Корневой компонент редактора |
| `BlockToolbar.vue` | Floating toolbar форматирования |
| `BlockControls.vue` | Контролы блока (drag, удалить, добавить) |
| `SlashCommandMenu.vue` | Меню slash-команд |
| `CollaboratorAvatars.vue` | Аватары соавторов в реальном времени |
| `blots/` | Quill blot-расширения для кастомных блоков |
| `embeds/` | Vue-компоненты для встраиваемых блоков (chart, code, image, SimpleTable, ...) |

### Blots

Каждый blot — Quill-класс, регистрирующий кастомный тип содержимого:

| Файл | Блок |
|------|------|
| `ImageBlot.js` | Изображение |
| `VideoBlot.js` | Видео |
| `CodeBlot.js` | Код |
| `MermaidBlot.js` | Mermaid-диаграмма |
| `ChartBlot.js` | График |
| `ChecklistBlot.js` | Чеклист |
| `ToggleBlot.js` | Сворачиваемый блок |
| `CalloutBlot.js` | Callout (информация/предупреждение) |
| `TabsBlot.js` | Вкладки |
| `ColumnsBlot.js` | Многоколоночный макет |
| `IntegramTableBlot.js` | Встроенная таблица Integram |
| `IntegramReportBlot.js` | Встроенный отчёт |
| `SimpleTableBlot.js` | Простая таблица |
| `RecordMentionBlot.js` | Упоминание записи |
| `FormulaBlot.js` | Формула (LaTeX) |
| `MapBlot.js` | Карта |
| `TimelineBlot.js` | Таймлайн |
| `ButtonBlot.js` | Кнопка-действие |
| `EmbedBlot.js` | Внешний embed (iframe) |
| `QuoteBlot.js` | Цитата |
| `DividerBlot.js` | Горизонтальный разделитель |
| `TocBlot.js` | Оглавление |
| `MentionBlot.js` | @-упоминание |
| `ProgressBlot.js` | Прогресс-бар |

## automations/

| Файл | Назначение |
|------|-----------|
| `SchedulePicker.vue` | Визуальный выбор cron-расписания |
| `TemplateInput.vue` | Ввод шаблона с переменными `{{fieldName}}` |
| `ConditionBuilder.vue` | Конструктор условий для триггеров и фильтров |
| `ActionCard.vue` | Карточка действия в редакторе автоматизации. `send_telegram`: режимы клавиатуры (Простой/Экраны). Простой режим — текст кнопки, поле, значение. Режим Экраны — многоэкранная клавиатура: список экранов (добавить/удалить/переименовать), редактор кнопок экрана (текст, тип go/url, цель), настройка начального экрана. Использует `useSchemaOptions` для каскадных пикеров (таблица → столбцы → записи) в конфигурации listSource/goFilter. |
| `SchemaPicker.vue` | Каскадный пикер таблица → колонки. v-model: `{ typeId, columns: { key: colId } }`. Props: `columnKeys` (Array — ключи колонок для выбора), `columnLabels` (Object — подписи). Загружает схему через `schemaService`. |
| `TgBotAutomationWizard.vue` | Диалог быстрого создания реакции Telegram-бота. Триггеры: команда (`on_telegram_command`) или ключевое слово (`on_telegram_message`, contains/exact). Генерирует автоматизацию с действием `send_telegram`. Props: `visible` (Boolean), `botId` (Number). Emits: `created`, `update:visible`. |

## files/

| Файл | Назначение |
|------|-----------|
| `DocReviewDialog.vue` | Диалог AI-разбора загруженного документа |

## dashboard/

| Файл | Назначение |
|------|-----------|
| `DashboardWidget.vue` | Обёртка виджета дашборда (resize, drag, header) |
| `widgets/ChartWidget.vue` | Виджет графика |
| `widgets/KpiWidget.vue` | Виджет KPI-метрики |
| `widgets/TableWidget.vue` | Виджет таблицы |
| `widgets/ReportWidget.vue` | Виджет отчёта |
| `widgets/TextWidget.vue` | Виджет текста |
| `widgets/KanbanWidget.vue` | Виджет канбана |
| `widgets/PivotWidget.vue` | Виджет сводной таблицы |
| `widgets/GalleryWidget.vue` | Виджет галереи |
| `widgets/DocumentWidget.vue` | Виджет документа |
| `widgets/CardWidget.vue` | Виджет карточки записи. Показывает одну запись в виде шапки (название + поля). Реагирует на crossFilter (`fieldId='_id'`). Поддерживает кросс-табличную связь через `config.linkedChildTypeId` — если crossFilter пришёл от дочерней таблицы, берёт `parentId` клика и загружает родительскую запись. Ref-колонки (type=1) резолвятся через `lookupsService` — показывает текст вместо сырых ID. |
| `widgets/FormWidget.vue` | Form for record creation/editing with automatic parent-table detection. Config: typeId, parentTypeId |
| `widgets/ActionWidget.vue` | Multi-mode action execution. Modes: automation (batch run), ai_button (run AI button on record), clear_table (delete all records) |
| `widgets/WMatrixWidget.vue` | W-matrix (ONA graph) widget. Shows collaboration graph between team members: hubs (green), bridges (orange), isolated (red). Spring-layout SVG rendering without D3 dependency. Config: roomId, days selector (7/30/90/180/365). |

## teamchat/

| File | Purpose |
|------|---------|
| `MessageCards.vue` | Inline decision cards rendered inside chat message bubbles. Shows decision verdict (Tag), domain (Chip), team members, and conflict pairs with router-links to decision detail page. |

## codespace/

| Файл | Назначение |
|------|-----------|
| `CodeEditor.vue` | Редактор кода (Monaco) |
| `PrDrawer.vue` | Боковая панель pull request. Diff: открытые PR — `getDiff(target, source)` (three-dot `git diff target...source`), влитые — `getCommitDiff(merge_commit_sha)` (для merge-коммитов использует `git diff sha^1 sha^2`) |
| `CommitDrawer.vue` | Боковая панель коммита |
| `DiffViewer.vue` | Просмотр diff |
| `GithubSyncPanel.vue` | Панель настройки GitHub Sync — конфигурация (URL, PAT, направление), webhook URL, ручной push/pull, отключение |
| `BlameView.vue` | git blame view with per-line author/SHA/timestamp grouping |
| `FileTable.vue` | GitHub-style file listing with commit messages and relative dates |
| `FileViewer.vue` | File content viewer with mode toggle, image preview, binary detection |
| `RepoHome.vue` | Repository landing: latest commit banner, FileTable, README rendering |
| `AboutSidebar.vue` | Repository sidebar: description, clone URL, commit/branch counts |
| `PathBreadcrumbs.vue` | Path breadcrumb navigation for repo browsing |

## reports/

| Файл | Назначение |
|------|-----------|
| `SubtotalsPanel.vue` | Панель промежуточных итогов отчёта |

## ai/

> **Note:** `CitationRenderer.vue`, `AgentSteps.vue`, `DebatePanel.vue` live in `components/` root, not `components/ai/`.

| Файл | Назначение |
|------|-----------|
| `AgentElicitMessage.vue` | Сообщение агента с запросом уточнения (HITL) |
| `CitationRenderer.vue` | Рендер markdown с inline-цитатами `[N]` (кликабельные бейджи) и структурированным списком источников (решения, топики, KAG, веб, таблицы, документы). Используется в Meta KB. Props: `text` (String). |
| `AgentSteps.vue` | Timeline-визуализация шагов агента: thinking-блоки и tool calls с человекочитаемыми статусами. Фильтрует ошибочные вызовы, сворачивает шаги (max 8), показывает expandable результаты. Используется в Meta KB. Props: `toolCalls` (Array), `streaming` (Boolean). |
| `DebatePanel.vue` | UI мультифазных дебатов экспертов: позиции (параллельный стриминг), кросс-дебаты (ответы), синтез (консенсус). Фазовый индикатор с прогрессом, grid/list переключатель, expandable позиции с tool calls. Используется в Meta KB. Props: `debate` (Object). |

## agent-suggestions/

| Файл | Назначение |
|------|-----------|
| `SuggestionInlineBlock.vue` | Блок предложений агентов — показывает pending-предложения с кнопками «Создать», «Изменить», «Отклонить», «Не нужно». Emits: `apply(s)`, `edit(s)`, `changed`. Монтируется в `AgentList.vue`. |
| `EditSuggestionDraftModal.vue` | Модалка редактирования черновика предложения агента. Props: `modelValue` (Boolean), `suggestion` (Object). Emits: `save({ suggestion, draft })`, `update:modelValue`. Поля: название, описание, примеры запросов (Chips), возможности (MultiSelect). |
| `SimilarSuggestionHint.vue` | Подсказка о похожих предложениях при создании агента. Props: `name` (String), `description` (String). Watches оба пропса с дебаунсом 500мс, ищет через `agentSuggestionsService.searchSimilar`. Emits: `use(suggestion)`. |

## portal/

| Файл | Назначение |
|------|-----------|
| `PortalConfigurator.vue` | Компонент конфигуратора портала |
| `editor/EditorLeftPanel.vue` | Левая панель визуального редактора портала |
| `editor/EditorRightPanel.vue` | Правая панель (настройки модуля) |
| `editor/EditorPreview.vue` | Предпросмотр портала |
| `editor/ColumnSelect.vue` | Выбор колонки для конфигурации модуля |
| `editor/ColorPicker.vue` | Выбор цвета |
| `editor/ImagePicker.vue` | Выбор изображения |
| `editor/ModuleField.vue` | Поле конфигурации модуля |
| `editor/EditorRightPanel.vue` (inline) | Мультивыбор ролей реализован inline внутри `EditorRightPanel.vue`, отдельного файла `RoleMultiSelect.vue` не существует |
| `editor/modules/HeroModule.vue` | Редактор hero-секции |
| `editor/modules/CatalogModule.vue` | Редактор каталога |
| `editor/modules/GalleryModule.vue` | Редактор галереи |
| `editor/modules/OrdersModule.vue` | Редактор заказов |
| `editor/modules/ProfileModule.vue` | Редактор профиля |
| `editor/modules/ReportsModule.vue` | Редактор отчётов |
| `editor/modules/AboutModule.vue` | Редактор страницы «О нас» |
| `editor/modules/AuthModule.vue` | Редактор авторизации |
| `editor/modules/LandingModule.vue` | Редактор лендинга |
| `editor/modules/FeaturedProductsModule.vue` | Редактор «Рекомендуемых товаров» |
| `editor/modules/GenericModule.vue` | Универсальный редактор модуля |
| `editor/modules/CustomCodeModule.vue` | Редактор кастомного кода: выбор codespace-репозитория, файла, ветки, библиотек, привязки данных (dropdown таблиц/отчётов) |
| `editor/modules/CustomCodePreview.vue` | Превью кастомного кода в визуальном редакторе: репозиторий, файл, ветка, подключённые библиотеки |
| `editor/panels/ChatConfigPanel.vue` | Панель конфигурации чата |
