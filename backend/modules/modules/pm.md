# PM — Модуль управления задачами

Полноценный модуль управления задачами внутри платформы Integram. Двухуровневая архитектура: задачи внутри каждого воркспейса (доска, бэклог, спринты) + организация как портфель проектов с кросс-проектной агрегацией.

## Ключевые принципы

- **Воркспейс = проект.** Внутри — задачи, спринты, доска. В сайдбаре раздел называется "Задачи".
- **Организация = портфель.** Объединяет воркспейсы. Показывает все задачи, команду, прогресс по проектам.
- **Участники организации = участники воркспейсов.** Нет отдельного механизма добавления в орг — участники подтягиваются из воркспейсов автоматически.
- **Изоляция данных.** Каждый воркспейс — отдельная PG-схема. Кросс-проектная видимость через org aggregation.
- **Satellite-таблицы, не EAV.** PM — dedicated модуль с собственными таблицами (`_v2_pm_issues`, `_v2_pm_sprints` и т.д.), lazy-init при первом обращении.

---

## Архитектура

### Backend

```
backend/src/api/v2/modules/pm/
  schema.js         — DDL для 9 satellite-таблиц, lazy-init
  service.js        — Issues CRUD, hierarchy, history, bulk ops
  sprints.js        — Sprint lifecycle (create, start, complete, auto-rollover)
  board.js          — Board/backlog data aggregation
  comments.js       — Issue comments
  links.js          — Issue-to-issue links (blocks, relates_to, duplicates)
  watchers.js       — Watch/unwatch issues
  data-links.js     — Issue ↔ workspace data links (table/document/report)
  templates.js      — Issue templates
  metrics.js        — Velocity, burndown, cycle time, lead time, CFD, workload
  listeners.js      — Codespace PR auto-link, notifications, automation triggers
  router.js         — HTTP endpoints

backend/src/api/v2/modules/orgs/
  pm-aggregation.js — Cross-workspace: задачи, команда, портфель
  pm-listeners.js   — Cache invalidation для org-level данных
```

### Frontend

```
frontend/src/
  services/pm.js              — API service (getDbApi pattern)
  stores/pm.js                — Pinia store
  views/pm/
    PmLayout.vue              — Layout с табами (Доска, Список, Бэклог, Спринт, Гант)
    BoardView.vue             — Kanban-доска с drag-drop
    ListView.vue              — Таблица с фильтрами
    BacklogView.vue           — Бэклог + создание спринтов
    SprintView.vue            — Прогресс спринта
    GanttView.vue             — Диаграмма Ганта (frappe-gantt)
  components/pm/
    IssueCard.vue             — Карточка на доске (цвет по приоритету)
    IssueDetail.vue           — Slide-over панель деталей задачи
    IssueForm.vue             — Диалог создания/редактирования
    IssueChecklist.vue        — Чек-лист внутри задачи
    RecurrencePicker.vue      — Выбор повторения задачи
    StatusBadge.vue           — Бейдж статуса
    PriorityBadge.vue         — Бейдж приоритета
    SprintHeader.vue          — Заголовок спринта с прогресс-баром
  views/orgs/
    OrgEdit.vue               — Дашборд организации (табы: Портфель, Задачи, Команда, Настройки)
    OrgPortfolio.vue          — Карточки проектов с прогрессом
    OrgMyIssues.vue           — Все задачи организации с фильтром
    OrgPeople.vue             — Команда (все участники воркспейсов)
    OrgSettings.vue           — Настройки + управление воркспейсами
    OrgWorkspaces.vue         — Привязка воркспейсов к организации
```

---

## База данных

9 satellite-таблиц, создаются lazy при первом обращении к PM в воркспейсе:

| Таблица | Назначение |
|---------|-----------|
| `_v2_pm_issues` | Задачи (title, type, status, priority, assignee, sprint, hierarchy, checklist, recurrence, start_date) |
| `_v2_pm_sprints` | Спринты (name, goal, dates, status) |
| `_v2_pm_issue_links` | Связи между задачами (blocks, relates_to, duplicates) |
| `_v2_pm_comments` | Комментарии к задачам |
| `_v2_pm_issue_history` | История изменений (field, old_value, new_value) |
| `_v2_pm_watchers` | Подписчики на задачу |
| `_v2_pm_data_links` | Привязки к данным воркспейса (таблицы, документы, отчёты) |
| `_v2_pm_issue_templates` | Шаблоны задач |
| `_v2_pm_issue_counter` | Атомарный счётчик для PM-{number} |

### Статусы задач

| Статус | Описание | Категория |
|--------|----------|-----------|
| `backlog` | В бэклоге, не запланировано | backlog |
| `todo` | Запланировано, не начато | todo |
| `in_progress` | В работе | in_progress |
| `in_review` | На ревью | in_progress |
| `done` | Завершено | done |
| `canceled` | Отменено | done |

### Типы задач

| Тип | Описание |
|-----|----------|
| `epic` | Группирующая задача, содержит tasks |
| `feature` | Новая функциональность |
| `task` | Конкретная работа |
| `bug` | Ошибка |

### Приоритеты (фиксированные)

`urgent` (красный) → `high` (оранжевый) → `medium` (жёлтый) → `low` (синий) → `none` (серый)

### Иерархия

3 уровня через `parent_id`: Epic → Task → Sub-task. Epic не может иметь parent. Sub-task не появляется на доске отдельно.

### Issue ID

Human-readable: `PM-1`, `PM-2`, ... Атомарный счётчик per workspace в таблице `_v2_pm_issue_counter`.

---

## API Endpoints

### Workspace-level (`/:db/pm/...`)

#### Задачи

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/issues` | Список задач (фильтры: status, type, priority, assignee_id, sprint_id, search, labels) |
| GET | `/issues/:id` | Детали задачи (+ children, links, comments) |
| POST | `/issues` | Создать задачу |
| PATCH | `/issues/:id` | Обновить задачу (пишет историю) |
| DELETE | `/issues/:id` | Soft delete (в корзину) |
| PATCH | `/issues/reorder` | Bulk reorder (drag-drop) |
| PATCH | `/issues/bulk` | Bulk update |
| DELETE | `/issues/bulk` | Bulk delete |

#### Комментарии

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/issues/:id/comments` | Список комментариев |
| POST | `/issues/:id/comments` | Добавить комментарий |
| PATCH | `/comments/:commentId` | Редактировать |
| DELETE | `/comments/:commentId` | Удалить |

#### Связи

| Метод | Путь | Описание |
|-------|------|----------|
| POST | `/issues/:id/links` | Создать связь (blocks, relates_to, duplicates) |
| DELETE | `/links/:linkId` | Удалить связь |

#### Наблюдатели

| Метод | Путь | Описание |
|-------|------|----------|
| POST | `/issues/:id/watch` | Подписаться |
| DELETE | `/issues/:id/watch` | Отписаться |
| GET | `/issues/:id/watchers` | Список подписчиков |

#### Привязки к данным

| Метод | Путь | Описание |
|-------|------|----------|
| POST | `/issues/:id/data-links` | Привязать запись/документ/отчёт |
| DELETE | `/data-links/:linkId` | Отвязать |
| GET | `/issues/:id/data-links` | Список привязок |

#### Спринты

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/sprints` | Список спринтов |
| POST | `/sprints` | Создать спринт |
| PATCH | `/sprints/:id` | Обновить |
| POST | `/sprints/:id/start` | Запустить (максимум 1 active) |
| POST | `/sprints/:id/complete` | Завершить (auto-rollover незакрытых) |
| DELETE | `/sprints/:id` | Удалить (только planning) |

#### Доска и бэклог

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/board` | Задачи по статусам |
| GET | `/backlog` | Задачи без спринта |

#### Шаблоны

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/templates` | Список шаблонов |
| POST | `/templates` | Создать |
| PATCH | `/templates/:id` | Обновить |
| DELETE | `/templates/:id` | Удалить |

#### Корзина

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/trash` | Удалённые задачи |
| POST | `/trash/:id/restore` | Восстановить |

#### Участники

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/members` | Участники воркспейса (для picker-ов) |

#### Метрики

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/metrics/velocity` | Story points за спринт (last N) |
| GET | `/metrics/burndown` | Remaining points per day |
| GET | `/metrics/cycle-time` | Среднее время in_progress → done |
| GET | `/metrics/lead-time` | Среднее время created → done |
| GET | `/metrics/cfd` | Cumulative Flow Diagram |
| GET | `/metrics/workload` | Загрузка по людям |

#### Экспорт

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/export` | CSV экспорт задач |

### Org-level (`/orgs/:slug/pm/...`)

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/my-issues` | Задачи из всех воркспейсов орга (фильтр: assignee=all/me/userId) |
| GET | `/people` | Команда — все участники всех воркспейсов + PM-статистика |
| GET | `/portfolio` | Прогресс по проектам (воркспейсам) |
| POST | `/issues` | Создать задачу в конкретном воркспейсе (`{ workspace: 'slug', ... }`) |

### Org workspace management (`/orgs/:slug/workspaces`)

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/` | Список воркспейсов организации |
| POST | `/` | Привязать воркспейс (`{ workspace: 'slug' }`) |
| DELETE | `/:ws` | Отвязать воркспейс |

---

## AI Tools

53 инструмента, доступны через `search_tools("pm")` в MCP. Полное покрытие HTTP API.

### Основные (CRUD)

| Инструмент | Tier | Описание |
|-----------|------|----------|
| `pm_list_issues` | LOW | Список задач с фильтрами |
| `pm_get_issue` | LOW | Детали задачи |
| `pm_create_issue` | MEDIUM | Создать задачу |
| `pm_update_issue` | MEDIUM | Обновить задачу |
| `pm_delete_issue` | HIGH | Удалить задачу |
| `pm_move_issue` | MEDIUM | Переместить (sprint, parent) |
| `pm_toggle_checklist` | MEDIUM | Переключить пункт чек-листа |
| `pm_bulk_update` | MEDIUM | Массовое обновление задач |
| `pm_bulk_delete` | HIGH | Массовое удаление задач (в корзину) |

### Комментарии

| Инструмент | Tier | Описание |
|-----------|------|----------|
| `pm_list_comments` | LOW | Список комментариев задачи |
| `pm_add_comment` | MEDIUM | Добавить комментарий |
| `pm_update_comment` | MEDIUM | Редактировать (только автор) |
| `pm_delete_comment` | HIGH | Удалить (только автор) |

### Связи и привязки

| Инструмент | Tier | Описание |
|-----------|------|----------|
| `pm_link_issues` | MEDIUM | Создать связь между задачами |
| `pm_unlink_issues` | MEDIUM | Удалить связь |
| `pm_link_data` | MEDIUM | Привязать задачу к данным (таблица/документ/отчёт) |
| `pm_unlink_data` | MEDIUM | Отвязать данные от задачи |
| `pm_list_data_links` | LOW | Список привязок задачи к данным |

### Наблюдатели

| Инструмент | Tier | Описание |
|-----------|------|----------|
| `pm_watch_issue` | MEDIUM | Подписаться на задачу |
| `pm_unwatch_issue` | MEDIUM | Отписаться |
| `pm_list_watchers` | LOW | Список подписчиков |

### Корзина и шаблоны

| Инструмент | Tier | Описание |
|-----------|------|----------|
| `pm_list_trash` | LOW | Удалённые задачи |
| `pm_restore_issue` | MEDIUM | Восстановить из корзины |
| `pm_list_templates` | LOW | Список шаблонов |
| `pm_create_template` | MEDIUM | Создать шаблон |
| `pm_update_template` | MEDIUM | Обновить шаблон |
| `pm_delete_template` | HIGH | Удалить шаблон |

### Участники и экспорт

| Инструмент | Tier | Описание |
|-----------|------|----------|
| `pm_list_members` | LOW | Участники воркспейса (для назначения) |
| `pm_export_csv` | LOW | Экспорт задач в CSV |

### Спринты

| Инструмент | Tier | Описание |
|-----------|------|----------|
| `pm_list_sprints` | LOW | Список спринтов |
| `pm_create_sprint` | MEDIUM | Создать спринт |
| `pm_update_sprint` | MEDIUM | Обновить |
| `pm_start_sprint` | HIGH | Запустить |
| `pm_complete_sprint` | HIGH | Завершить (auto-rollover) |
| `pm_delete_sprint` | HIGH | Удалить |

### Доска и метрики

| Инструмент | Tier | Описание |
|-----------|------|----------|
| `pm_get_board` | LOW | Канбан-доска |
| `pm_get_backlog` | LOW | Бэклог |
| `pm_get_sprint_progress` | LOW | Прогресс спринта |
| `pm_get_velocity` | LOW | Velocity |
| `pm_get_burndown` | LOW | Burndown |
| `pm_get_cycle_time` | LOW | Cycle time |
| `pm_get_lead_time` | LOW | Lead time |
| `pm_get_workload` | LOW | Загрузка команды |

### AI-powered

| Инструмент | Tier | Описание |
|-----------|------|----------|
| `pm_detect_blockers` | LOW | Найти заблокированные/просроченные/застрявшие задачи |
| `pm_summarize_sprint` | LOW | Сводка спринта |
| `pm_triage_issue` | LOW | Авто-классификация: тип, приоритет, метки |
| `pm_decompose_issue` | LOW | Декомпозиция на подзадачи |
| `pm_suggest_estimate` | LOW | Предложить оценку (историческая медиана или эвристика) |
| `pm_plan_sprint` | MEDIUM | Планирование спринта по приоритетам и velocity |

### Org-level

| Инструмент | Tier | Описание |
|-----------|------|----------|
| `pm_org_my_issues` | LOW | Задачи по всем проектам орга |
| `pm_org_workload` | LOW | Загрузка команды по всем проектам |
| `pm_org_portfolio` | LOW | Портфель проектов |
| `pm_org_create_issue` | MEDIUM | Создать задачу через орг |

---

## Интеграции

### Codespace (Git/PR)

Naming convention `PM-{number}` в branch/PR/commit:

- PR created с `PM-42` → задача переходит в `in_review` (если была `in_progress`)
- PR merged → задача переходит в `done`

Реализация: `listeners.js` слушает `codespace.pr.created` и `codespace.pr.merged`.

### WebSocket

Канал `pm` — real-time обновления:

| Событие | Описание |
|---------|----------|
| `pm:issue-created` | Новая задача |
| `pm:issue-updated` | Обновление |
| `pm:issue-status-changed` | Смена статуса |
| `pm:issue-deleted` | Удаление |
| `pm:comment-added` | Новый комментарий |
| `pm:sprint-started` | Спринт запущен |
| `pm:sprint-completed` | Спринт завершён |

### Уведомления

- Назначение исполнителя → уведомление назначенному
- Смена статуса → уведомление наблюдателям
- Новый комментарий → уведомление наблюдателям

Автоматические наблюдатели: reporter + assignee.

### Automations

PM события доступны как триггеры автоматизаций: `on_issue_created`, `on_issue_status_changed`, `on_sprint_started`, `on_sprint_completed`.

---

## Спринты

Опциональные. Если не создал ни одного — работаешь как Kanban (доска + бэклог). Создал спринт — переключаешься в Scrum.

### Жизненный цикл

1. `planning` — создан, задачи добавляются
2. `active` — запущен (максимум 1 per workspace)
3. `completed` — завершён

### Завершение спринта

- Задачи `done`/`canceled` → остаются в спринте (для velocity)
- Незакрытые → переносятся в следующий `planning` спринт (если есть) или в бэклог

---

## Frontend

### Доска (BoardView)

Kanban с 5 колонками: Бэклог, К выполнению, В работе, На ревью, Готово. Drag-drop между колонками меняет статус. Фильтры: поиск, тип, приоритет, исполнитель.

Карточки цветные по приоритету (левый бордер): красный → оранжевый → жёлтый → синий.

### Список (ListView)

DataTable с сортировкой по всем колонкам. Поиск и фильтр по статусу. Inline-редактирование статуса, приоритета и исполнителя через Select dropdown — изменения сохраняются мгновенно при выборе.

### Бэклог (BacklogView)

Задачи без спринта. Кнопка "+" для создания спринта. Кнопка "→" для переноса в активный спринт. Drag-to-reorder: перетаскивание задач меняет порядок (HTML5 DnD, `PATCH /pm/issues/reorder`). Статус спринта локализован (Планирование/Активный/Завершён).

### Спринт (SprintView)

Прогресс-бар, кнопки "Запустить" / "Завершить" с подтверждением. Колонки спринтовой доски локализованы: К выполнению, В работе, На ревью, Готово.

### Детали задачи (IssueDetail)

Slide-over панель справа. Поля: статус, приоритет, исполнитель, тип, спринт, оценка (SelectButton 1-21), дедлайн, описание (Markdown), подзадачи, комментарии. Кнопка удаления.

### Sidebar

Collapsible секция "ЗАДАЧИ" с подпунктами: Доска, Список, Бэклог, Спринт, Гант. Каждый пункт — `router-link` с `?tab=` query параметром.

**Active state:** Ручная привязка `:class="{ active: ... }"` вместо `active-class` — стандартный `active-class` у Vue Router игнорирует query params, поэтому все ссылки с одинаковым path подсвечивались одновременно.

**Bidirectional tab↔URL sync (PmLayout.vue):**
- URL → tab: `watch(route.query.tab)` синхронизирует `activeTab` при навигации, back/forward, sidebar clicks
- Tab → URL: `watch(activeTab)` вызывает `router.replace()` при клике на TabMenu
- Для вкладки "Доска" (default) `?tab=` опускается — чистый URL без query param

### Календарь (CalendarView)

Месячный грид с задачами по `due_date`/`start_date`. Навигация между месяцами. Задачи показаны как цветные чипы (цвет по приоритету). Клик по задаче открывает детали. Сегодняшний день выделен.

### Горячие клавиши

Composable `usePmShortcuts.js`. Работает только вне полей ввода (input/textarea/select).

| Клавиша | Действие |
|---------|----------|
| `C` / `С` | Открыть форму создания задачи |
| `1`-`6` | Переключить вкладку (Доска, Список, Бэклог, Спринт, Гант, Календарь) |

---

## Организация (Org-level)

### Портфель

Карточки проектов (воркспейсов) с прогресс-баром, счётчиками (всего, готово, в работе, просрочено), информацией об активном спринте.

### Задачи

Таблица ВСЕХ задач организации из ВСЕХ воркспейсов. Колонки: Проект, ID, Название, Исполнитель, Статус, Приоритет, Баллы, Срок. Фильтр: Все / Мои.

### Команда

Все участники всех воркспейсов организации (подтягиваются из `_v2_memberships`). Метрики: активных задач, в работе, просрочено, баллы, проекты.

### Настройки

Название, slug организации. Управление воркспейсами: добавить (из существующих), убрать.

### Навигация

- Global sidebar: Рабочий стол → Организации → Мои рабочие области
- Workspace switcher dropdown: кнопка "Организации"
- Login screen: кнопка "Организации"

---

## Кэширование

Org-level запросы кэшируются в Valkey (Redis):
- TTL: 60 секунд
- Prefix: `pm_org:`
- Инвалидация: event-driven через `pm.issue.created/updated/deleted`, `pm.sprint.completed`
- Ключи включают userId (разные пользователи видят разные воркспейсы)
- Ключи отслеживаются через SMEMBERS (не KEYS)

---

## Файлы

### Backend (14 файлов)

| Файл | Строк | Описание |
|------|-------|----------|
| `schema.js` | ~150 | DDL 9 таблиц + 10 индексов |
| `service.js` | ~340 | Issues CRUD, hierarchy, history, bulk |
| `sprints.js` | ~150 | Sprint lifecycle |
| `board.js` | ~55 | Board/backlog aggregation |
| `comments.js` | ~65 | Comments CRUD |
| `links.js` | ~35 | Issue-to-issue links |
| `watchers.js` | ~30 | Watch/unwatch |
| `data-links.js` | ~40 | Data links |
| `templates.js` | ~45 | Issue templates |
| `metrics.js` | ~160 | 6 metric functions |
| `listeners.js` | ~130 | PR auto-link, notifications, automations |
| `router.js` | ~340 | HTTP endpoints |
| `recurring.js` | ~70 | Cron job для повторяющихся задач |

### Frontend (12 файлов)

| Файл | Описание |
|------|----------|
| `services/pm.js` | API service |
| `stores/pm.js` | Pinia store |
| `views/pm/PmLayout.vue` | Layout с табами |
| `views/pm/BoardView.vue` | Kanban-доска |
| `views/pm/ListView.vue` | Таблица |
| `views/pm/BacklogView.vue` | Бэклог |
| `views/pm/SprintView.vue` | Спринт |
| `views/pm/GanttView.vue` | Диаграмма Ганта |
| `views/pm/CalendarView.vue` | Месячный календарь по дедлайнам |
| `composables/usePmShortcuts.js` | Горячие клавиши PM |
| `components/pm/IssueCard.vue` | Карточка |
| `components/pm/IssueDetail.vue` | Детали |
| `components/pm/IssueForm.vue` | Форма создания |
| `components/pm/IssueChecklist.vue` | Чек-лист |
| `components/pm/RecurrencePicker.vue` | Повторение |
| `components/pm/StatusBadge.vue` | Бейдж статуса |
| `components/pm/PriorityBadge.vue` | Бейдж приоритета |
| `components/pm/SprintHeader.vue` | Заголовок спринта |

### AI Tools (в `ai/agent/tools/pm.js`)

53 tool handler функции, TOOL_DEFS в `agent/index.js`, cases в `tool-executor.js`, тиры в `risk-tiers.js`. Полное покрытие HTTP API модуля.

### Тесты (11 файлов, 28 тестов)

```
backend/src/api/v2/modules/pm/__tests__/
  schema.test.js, service.test.js, sprints.test.js, board.test.js,
  sub-services.test.js, integration.test.js, listeners.test.js, metrics.test.js

backend/src/api/v2/modules/orgs/__tests__/
  pm-aggregation.test.js, pm-integration.test.js

backend/src/api/v2/modules/ai/agent/__tests__/
  pm-tools.test.js
```

---

## Диаграмма Ганта

Визуализация задач на временной шкале. Использует `frappe-gantt` (SVG, open-source).

- **Режимы**: Day, Week, Month
- **Цвет баров**: по приоритету (красный/оранжевый/жёлтый/синий)
- **Drag-and-drop**: перетаскивание дат обновляет `start_date` и `due_date` через API
- **Фильтр**: показывает только задачи с `start_date` или `due_date`
- **Прогресс**: done=100%, in_review=75%, in_progress=50%, остальные=0%
- **Dark mode**: поддерживается через CSS custom properties

Для работы Ганта необходимы поля `start_date` и/или `due_date` на задачах.

---

## Чек-листы

Простые галочки внутри задачи — не подзадачи, а пункты для проверки.

- **Хранение**: JSONB колонка `checklist` в `_v2_pm_issues`, формат `[{text: string, done: boolean}]`
- **API**: `PATCH /:db/pm/issues/:id/checklist` — toggle пункта по индексу
- **UI**: секция "Чек-лист (N/M)" в IssueDetail с чекбоксами, добавлением и удалением пунктов
- **Повторяющиеся задачи**: при создании копии чек-лист сбрасывается (все `done` → `false`)

---

## Повторяющиеся задачи

Автоматическое создание копий задач по расписанию.

- **Типы**: `daily`, `weekly`, `biweekly`, `monthly`
- **Хранение**: колонка `recurrence` (VARCHAR) в `_v2_pm_issues`
- **Логика**: cron job (`recurring.js`) каждый час проверяет все воркспейсы
- **Триггер**: задача с `recurrence` перешла в `done`/`canceled` → создаётся новая копия в `backlog`
- **Защита от дублей**: `NOT EXISTS` проверяет нет ли открытой копии с тем же title+recurrence
- **Чек-лист**: при создании копии все пункты сбрасываются в `done: false`
- **UI**: dropdown "Повторение" в форме создания задачи (Не повторяется / Ежедневно / Еженедельно / Раз в 2 недели / Ежемесячно)

