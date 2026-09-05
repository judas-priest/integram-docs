# PM — Модуль управления задачами

Полноценный модуль управления задачами внутри платформы Integram. Двухуровневая архитектура: задачи внутри каждого воркспейса (доски, бэклог, спринты) + организация как портфель проектов с кросс-проектной агрегацией.

## Ключевые принципы

- **Воркспейс = проект.** Внутри — задачи, спринты, доска. В сайдбаре раздел называется "Задачи".
- **Организация = портфель.** Объединяет воркспейсы. Показывает все задачи, команду, прогресс по проектам.
- **Участники организации = участники воркспейсов.** Вкладка «Команда» подтягивает людей из воркспейсов автоматически, отдельного списка у неё нет. Но членство в самой организации существует (`_v2_memberships.org_id`) и само по себе открывает доступ к сводкам PM — см. `orgs.md`.
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
  boards.js         — Доски (воронки) и CRUD статусов доски
  statuses.js       — Реестр статусов с kind (набор — у каждой доски)
  comments.js       — Issue comments
  links.js          — Issue-to-issue links (blocks, relates_to, duplicates)
  watchers.js       — Watch/unwatch issues
  data-links.js     — Issue ↔ workspace data links (table/document/report/object) + обратный поиск
  templates.js      — Issue templates
  views.js          — Настройки вида (какие поля видны на карточках/колонках)
  metrics.js        — Velocity, burndown, cycle time, lead time, CFD, workload
  listeners.js      — Codespace PR auto-link, notifications, automation triggers
  recurring.js      — Cron job для повторяющихся задач
  csv.js            — CSV-экспорт
  router.js         — HTTP endpoints

backend/src/api/v2/modules/orgs/
  pm-aggregation.js — Cross-workspace: задачи, команда, портфель
  pm-listeners.js   — Cache invalidation для org-level данных
  role-level.js     — Уровень роли в SQL и обратно (общий с service.js)
```

### Frontend

```
frontend/src/
  services/pm.js              — API service (getDbApi pattern)
  stores/pm.js                — Pinia store
  views/pm/
    PmLayout.vue              — Layout с табами (Доска, Список, Бэклог, Спринт, Гант, Календарь, Метрики)
    BoardView.vue             — Kanban-доска с drag-drop
    ListView.vue              — Таблица с фильтрами
    BacklogView.vue           — Бэклог + создание спринтов
    SprintView.vue            — Прогресс спринта
    GanttView.vue             — Диаграмма Ганта (frappe-gantt)
    CalendarView.vue          — Месячный календарь по датам
    MetricsView.vue           — Скорость, сгорание, CFD, время цикла и поставки, загрузка
  components/pm/
    IssueCard.vue             — Карточка на доске (цвет по приоритету)
    PmChart.vue               — Обёртка над chart.js для экрана метрик
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

11 satellite-таблиц, создаются lazy при первом обращении к PM в воркспейсе:

| Таблица | Назначение |
|---------|-----------|
| `_v2_pm_boards` | Доски (воронки): name, sort |
| `_v2_pm_issues` | Задачи (title, type, status, priority, assignee, sprint, board, hierarchy, checklist, recurrence, start_date) |
| `_v2_pm_views` | Настройки вида (карточки/колонки): config JSONB, scope personal/shared, is_default |
| `_v2_pm_sprints` | Спринты (name, goal, dates, status) |
| `_v2_pm_issue_links` | Связи между задачами (blocks, relates_to, duplicates) |
| `_v2_pm_comments` | Комментарии к задачам |
| `_v2_pm_issue_history` | История изменений (field, old_value, new_value) |
| `_v2_pm_watchers` | Подписчики на задачу |
| `_v2_pm_data_links` | Привязки к данным воркспейса (таблицы, записи EAV, документы, отчёты) |
| `_v2_pm_issue_templates` | Шаблоны задач |
| `_v2_pm_issue_counter` | Атомарный счётчик для PM-{number} |

### Статусы задач

Стартовый набор ниже сеется на доску «Основная»; у каждой доски области свой набор
(см. «Доски и воронки»).

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
| POST | `/issues/:id/data-links` | Привязать таблицу/запись/документ/отчёт (editor, CSRF) |
| DELETE | `/data-links/:linkId` | Отвязать (editor, CSRF) |
| GET | `/issues/:id/data-links` | Список привязок задачи |
| GET | `/issues/by-target?target_type=&target_id=` | Обратный поиск: задачи, привязанные к цели (маршрут объявлен выше `/issues/:id`) |

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

#### Доски и воронки

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/boards` | Доски области с числом живых задач |
| POST | `/boards` | Создать доску (`{ name, sort? }`; по умолчанию сеется копией статусов первой доски, `copyStatusesFrom` — источник, `empty: true` — пустая) |
| PATCH | `/boards/:id` | Переименовать / сменить sort |
| DELETE | `/boards/:id?moveTo=` | Удалить (ограничения ниже) |
| GET | `/statuses?board_id=` | Статусы: весь перечень области или одной доски |
| GET | `/boards/:id/statuses` | Статусы доски |
| POST | `/boards/:id/statuses` | Добавить статус (`{ name, label, kind, sort? }`) |
| PATCH | `/statuses/:statusId` | Переименовать / сменить sort / сменить kind |
| DELETE | `/statuses/:statusId` | Удалить статус |
| POST | `/issues/:id/move-board` | Перенести задачу на другую доску (`{ boardId }`) |

Портал: `GET /:db/portal/api/pm/statuses?board_id=N` (та же служба `listStatuses`, ответ `{ok:true, data:{items:[…]}}`, доступ под портальной кукой `portalAuth('*')`). Портальный TaskBoard читает отсюда живые колонки и метки; перечень недоступен — виджет показывает канонические статусы (фолбэк).

#### Шаблоны

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/templates` | Список шаблонов |
| POST | `/templates` | Создать |
| PATCH | `/templates/:id` | Обновить |
| DELETE | `/templates/:id` | Удалить |

#### Настройка карточек (вид)

| Метод | Путь | Описание | Права |
|-------|------|----------|-------|
| GET | `/views/display?board_id=` | Разрешённая настройка карточек: `{ source, config, boardDefault }` | любой участник |
| PUT | `/views/display` | Сохранить личную настройку (`{ board_id, config }`) | любой участник (включая viewer) |
| POST | `/views/default` | Поставить дефолт доски для всех (`{ board_id, config }`) | admin |
| DELETE | `/views/default?board_id=` | Снять дефолт доски | admin |

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
| GET | `/my-issues` | Задачи из воркспейсов орга (фильтры: assignee=all/me/userId, status, workspace=slug\|db_name). Пагинация: offset (`page`/`pageSize`) или keyset (`cursor`); на полной странице отдаётся `nextCursor` — с него начинается курсорная пагинация |
| GET | `/people` | Команда — все участники всех воркспейсов + PM-статистика |
| GET | `/portfolio` | Прогресс по проектам (воркспейсам) |
| GET | `/activity` | Лента активности из истории задач областей |
| GET | `/metrics` | Сводные метрики: velocity, cycle time, lead time |
| GET | `/suggest-assignee` | Подсказка исполнителя по нагрузке |
| GET/PUT | `/defaults` | Эталон PM организации: типы и статусы по умолчанию (чтение / запись) |
| POST | `/issues` | Создать задачу в конкретном воркспейсе (`{ workspace: 'slug', ... }`) |

### Org workspace management (`/orgs/:slug/workspaces`)

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/` | Список воркспейсов организации |
| POST | `/` | Привязать воркспейс (`{ workspace: 'slug' }`) |
| DELETE | `/:ws` | Отвязать воркспейс |

---

## AI Tools

56 инструментов, доступны через `search_tools("pm")`. Полное покрытие HTTP API.

Запрос по имени группы отдаёт все 56: имя группы освобождено от потолка выдачи
(`agent/tool-search.js`, `keep = Math.max(RESULT_CAP, groupExact)`). Расплывчатый
запрос — например `search_tools("задачи спринт")` — срезается на 30, и тогда ответ
несёт `_truncated: true` и `_hint` с числом срезанного по группам. Молчаливого среза
больше нет: до 23.08.2026 запрос отдавал ровно 30 и обрывался на `pm_list_sprints`,
оставляя за бортом управление спринтами, доску, бэклог, всю аналитику и шесть
AI-инструментов.

### Основные (CRUD)

| Инструмент | Tier | Описание |
|-----------|------|----------|
| `pm_list_issues` | LOW | Список задач с фильтрами |
| `pm_get_issue` | LOW | Детали задачи |
| `pm_create_issue` | MEDIUM | Создать задачу (опциональный `board_id` — доска; умолчание — первая по sort; статус валидируется внутри доски); по умолчанию назначает звавшего (`assigneeId: null` — явно без исполнителя) |
| `pm_update_issue` | MEDIUM | Обновить задачу |
| `pm_delete_issue` | HIGH | Удалить задачу |
| `pm_move_issue` | MEDIUM | Переместить (sprint, parent) |
| `pm_toggle_checklist` | MEDIUM | Переключить пункт чек-листа |
| `pm_bulk_update` | MEDIUM | Массовое обновление задач |
| `pm_bulk_delete` | HIGH | Массовое удаление задач (в корзину) |

Все HIGH-тулы PM (`pm_delete_issue`, `pm_bulk_delete`, `pm_delete_comment`,
`pm_delete_template`, `pm_delete_sprint`) исполняются только после
HITL-подтверждения — как orgs-деструктив: ответ `pending_confirmation`, резюм
через switch подтверждений в `ai/agent/index.js`.

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
| `pm_link_data` | MEDIUM | Привязать задачу к данным (таблица/запись EAV/документ/отчёт) |
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

### Доски и воронки

| Инструмент | Tier | Описание |
|-----------|------|----------|
| `pm_list_boards` | LOW | Доски области с числом задач |
| `pm_list_statuses` | LOW | Статусы области с kind; опциональный `board_id` — только одной доски |
| `pm_move_to_board` | MEDIUM | Перенос задачи на другую доску (`{ id, boardId }`); статус маппится по kind, ответ сообщает маппинг |

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
| `pm_org_my_issues` | LOW | Задачи по проектам орга; `workspace` сужает до одного |
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

Только в шине (без WS-канала, подписчики — future): `pm.comment.updated`,
`pm.comment.deleted` — правка и удаление комментария.

### Уведомления

- Назначение исполнителя → уведомление назначенному
- Смена статуса → уведомление наблюдателям
- Новый комментарий → уведомление наблюдателям

Автоматические наблюдатели: reporter + assignee.

### Best practices: создание задач (для агентов)

- Один атом работы → одна задача. Многокомпонентная работа → **эпик + дочерние
  задачи** (`parent_id`, глубина ≤3; эпик родителя не имеет), а не одна задача
  с простынёй описания.
- Тип выводится из формулировки: падает/ошибка → `bug`, «хочу/добавь» →
  `feature`, крупное без краёв → `epic`, иначе `task`.
- Срочность живёт в поле `priority` (urgent/high/medium/low/none), НЕ в метках.
- Метки — самоясные слова, отвечающие «откуда и про что»: источник («спека»),
  область (backend/frontend/platform), тема («pm», «orgs»). Без криптикодов
  (p0/p1 непонятны вне контекста). 2–4 метки. PATCH меток заменяет массив
  ЦЕЛИКОМ — при добавлении сначала прочитать текущие.
- Оценка (1-2-3-5-8-13-21) — только для ОТКРЫТЫХ задач; закрытым не ставят.
- `due_date` — только если дату назвал пользователь, не выдумывать.
- `done` ставится переходом статуса через update (closed_at выставляет
  сервис); создание сразу со статусом done ломает lead/cycle time.
- Вопросы пользователю — только когда формулировка подходит и под PM-задачу,
  и под другое (EAV-запись/документ), либо разбивка массового запроса (>3
  задач) неочевидна. В остальных случаях — выводить из контекста.

### Automations

PM события доступны как триггеры автоматизаций: `on_issue_created`, `on_issue_status_changed`, `on_sprint_started`, `on_sprint_completed`, `on_issue_commented`, `on_issue_updated`, `pm_deadline`. Роботы умеют писать в PM действиями `create_issue` и `update_issue` (см. `automations.md`).

### Роботы и дедлайны

`pm_deadline` — ежедневный скан задач по `due_date`. Автоматизация создаётся через штатный редактор автоматизаций: в настройках триггера задаются «За N дней» (`config.days`, умолчание 1 — горизонт до срока) и «Час напоминания» (`config.hour`, умолчание 7 UTC — час ежедневного скана).

Механика:

- Сканер (`automations/pm-deadline.js`) выбирает незакрытые задачи (kind статуса не `done`/`canceled` — по СВОЕЙ доске) с `due_date <= сегодня + N дней` и запускает робота по каждой.
- Дедуп — журнал `_v2_pm_deadline_log` с `UNIQUE(automation_id, issue_id, sent_on)`: одна задача срабатывает не чаще одного раза в день, даже при повторных сканах (фолбэк-планировщик без Redis гоняет скан чаще — это безвредно).
- Джоба живёт в BullMQ под ключом `pm_deadline:<db>:<automationId>`; ставится при сохранении включённой автоматизации и снимается при выключении/удалении — рестарт воркера не нужен.

Переменные `{{pm_*}}` доступны во всех pm-триггерах (`pm_deadline`, `on_issue_commented`, `on_issue_updated`, а также событийных): `pm_issue_id`, `pm_number`, `pm_title`, `pm_status`, `pm_due_date`, `pm_assignee_id`, `pm_assignee_email`, `pm_assignee_username`, у событий комментария — `pm_comment_id`.

Типовое напоминание исполнителю: действие `send_notification` с `username: '{{pm_assignee_email}}'` (адресация только по username/email), `title: 'Дедлайн по «{{pm_title}}» (PM-{{pm_number}})'`, `body: 'Срок истекает {{pm_due_date}}'`.

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

## Доски и воронки

В области может быть несколько досок (воронок): у каждой — собственный набор статусов, каждая задача живёт на одной доске.

### Модель

- `_v2_pm_boards` (`name`, `sort`) — сами доски.
- `_v2_pm_statuses.board_id` — набор статусов принадлежит доске; `UNIQUE(board_id, name)` — имя уникально ВНУТРИ доски, у двух досок могут быть одноимённые статусы с разными kind. Отсюда: join статусов всегда доску-осведомлённый (`statusJoin` в `statuses.js`), иначе «Готово» задачи доски 2 соединилось бы с одноимённым статусом доски 1.
- `_v2_pm_issues.board_id` — доска задачи; подзадача наследует доску родителя.

`kind` — машинная категория статуса (`open`/`active`/`done`/`canceled`). Все фильтры «завершённости» (closed_at, burndown, cycle/lead time, CFD, скрытие отменённых) идут по kind, а не по именам: имена пользовательские.

### Миграция при первом заходе

`ensureStatusSeed` (`schema.js`) идемпотентен и вызывается при каждом обращении к PM:

- создаёт `_v2_pm_boards`; на живой таблице статусов переносит `UNIQUE(name)` → `UNIQUE(board_id, name)` (DO-блок с ограждением по `pg_constraint` — у `ADD CONSTRAINT` нет `IF NOT EXISTS`);
- добавляет `board_id` в статусы и задачи;
- при пустом перечне досок заводит «Основная» (sort 10) и сеет на неё прежние 6 статусов;
- привязывает к первой доске статусы и задачи без `board_id`.

### Перенос задачи

`POST /issues/:id/move-board` `{ boardId }` → `moveIssueToBoard` (`service.js`):

- статус маппится по kind — берётся первый по sort статус той же категории в целевой доске; в ответе `fromStatus`/`status`, фронт показывает маппинг тостом;
- в целевой доске нет статуса нужной категории → 409; перенос на ту же доску → 400;
- `board_id`, статус и `closed_at` (CASE по kind ОБОИХ статусов: вход в закрытую категорию ставит `NOW()`, выход снимает) — одним UPDATE в одной транзакции с двумя строками истории (`board_id` и `status` — метрики читают обе);
- спринт не трогается — перенос между досками не выводит задачу из спринта;
- `pm.issue.updated` уходит ПОСЛЕ коммита (подписчик читает с другого соединения).

### Ограничения (по Azure DevOps)

- после ЛЮБОЙ операции в доске остаётся минимум один статус каждой из 4 категорий — проверяется и при удалении статуса, и при смене kind (`kindCategoriesOk`);
- максимум 32 статуса на доску;
- коллизия имени внутри доски — 409 (`UNIQUE(board_id, name)`, ловится явно до INSERT);
- последняя доска несгораемая (400);
- доска с задачами удаляется только через `?moveTo=<boardId>` — каждая переносится тем же kind-маппингом (DRY с ручным переносом); без `moveTo` — 409 с числом задач. Статусы удаляемой доски стираются ВМЕСТЕ с ней — иначе оставались бы сироты с `board_id` в никуда. Переносы идут до транзакции (каждый атомарен сам по себе), стирание статусов и доски — одной сделкой;
- новая доска по умолчанию сеется КОПИЕЙ набора статусов первой доски: пустая доска не показывает колонок, и первая же задача на ней получила бы статус, которого в её статусах нет (`empty: true` — осознанный выбор пустой).

### Решение: спринт и метрики — общие на воркспейс

Спринты и метрики не делятся по доскам: `sprint_id` у задачи не зависит от `board_id`, velocity/burndown/CFD/workload считают по всей области. Доска — только взгляд на поток, а не разбиение команды.

### Фронтенд

Активная доска выбирается селектором в шапке PM (`PmLayout.vue`, выбор помнится в localStorage по слагу воркспейса); при смене перечитываются статусы и данные вкладки. Список, Гант и Календарь показывают задачи активной доски (store грузит списки с `board_id` активной); спринт, бэклог и метрики от выбора доски не зависят. Управление статусами доски — кнопка у селектора.

---

## Настройка карточек (view options)

Пользователь выбирает, какие поля видны на карточках доски и в списке задач. Кнопка-шестерёнка (`pi-sliders-h`) в ряду фильтров BoardView и ListView открывает панель тумблеров; настройка хранится на сервере (сервер — источник правды, localStorage не используется) и переживает перезагрузку.

### Хранение

Таблица `_v2_pm_views` (создаётся тем же lazy-init, что остальные PM-таблицы):

| Колонка | Назначение |
|---------|-----------|
| `board_id` | Доска, к которой относится настройка |
| `owner` | Пользователь (для `scope='personal'`) |
| `name` | Всегда `__display__` (личная) или `__default__` (дефолт доски) — фундамент под будущие именованные виды |
| `config` | JSONB — карта «ключ поля → boolean» |
| `scope` | `personal` (личная, автосохранение) / `shared` (дефолт доски) |
| `is_default` | TRUE у единственного дефолта доски |

Уникальный индекс `UNIQUE(board_id, owner, name, scope)` — одна личная строка на пользователя и доску; частичный индекс `pm_views_board_default ON (board_id) WHERE scope='shared' AND is_default` — один дефолт на доску.

### Ключи config

| Ключ | Поле | Доска | Список |
|------|------|:-----:|:------:|
| `assignee` | Исполнитель | ✓ | ✓ |
| `labels` | Метки | ✓ | — |
| `estimate` | Баллы | ✓ | ✓ |
| `dueDate` | Срок | ✓ | ✓ |
| `subtasks` | Подзадачи (шеврон) | ✓ | — |
| `created` | Дата создания (относительное время, абсолютное в тултипе) | ✓ | ✓ |

Неизвестные ключи и не-boolean значения отбрасываются при записи и чтении (`sanitize`) — мусор из БД до карточек не доходит. Лимит размера — 4 КБ, большее — 400. На доске `subtasks` неприменим и в списке не хранится.

### Прецедент разрешения (клиент)

Личная настройка > дефолт доски (`scope='shared', is_default`) > встроенное. `GET /views/display` возвращает `{ source: 'personal'|'default'|'builtin', config, boardDefault }` — вместе с дефолтом доски, чтобы клиент мог предложить «вернуться к общему». «Сбросить к дефолту доски» отдельного маршрута не имеет: клиент пишет конфиг дефолта как личный (PUT), и после перечитывания источник снова `default`.

Личная настройка сохраняется автосохранением (debounce 500 мс после тумблера) — PUT доступен любому участнику, включая viewer: настройка не меняет данные доски. Командный дефолт — явное действие: `POST /views/default` и `DELETE /views/default` только для admin (`requireRole('admin')`), не-админ получает 403. Фронт дополнительно прячет кнопки «Дефолт для всех» у не-админов по роли из `stores/workspace.js` (`isAdmin` — из `myRole`/`role` в списке воркспейсов); 403 остаётся заслоном на сервере.

---

## Frontend

### Доска (BoardView)

Kanban с 5 колонками: Бэклог, К выполнению, В работе, На ревью, Готово. Drag-drop между колонками меняет статус. Фильтры: поиск, тип, приоритет, исполнитель.

Карточки цветные по приоритету (левый бордер): красный → оранжевый → жёлтый → синий.

Цвет срока (`dueDateClass`, `frontend/src/utils/dateInput.js`) — три состояния: просрочен / истекает сегодня / в норме; применяется и на карточке доски, и в списке.

### Список (ListView)

DataTable с сортировкой по всем колонкам. Поиск и фильтр по статусу. Inline-редактирование статуса, приоритета и исполнителя через Select dropdown — изменения сохраняются мгновенно при выборе.

### Бэклог (BacklogView)

Задачи без спринта. Кнопка "+" для создания спринта. Кнопка "→" для переноса в активный спринт. Drag-to-reorder: перетаскивание задач меняет порядок (HTML5 DnD, `PATCH /pm/issues/reorder`). Статус спринта локализован (Планирование/Активный/Завершён).

### Спринт (SprintView)

Прогресс-бар, кнопки "Запустить" / "Завершить" с подтверждением. Колонки спринтовой доски локализованы: К выполнению, В работе, На ревью, Готово.

### Детали задачи (IssueDetail)

Slide-over панель справа. Поля: статус, приоритет, исполнитель, тип, спринт, доска (Select — перенос на другую доску, маппинг статуса показывается тостом), оценка (SelectButton 1-21), дедлайн, описание (Markdown), подзадачи, комментарии. Кнопка удаления.

### Метрики (MetricsView)

Все шесть метрик одним экраном. Сверху четыре плитки: средняя скорость, время цикла (`in_progress → done`) и сквозное время (`created → done`) в днях со средним и медианой, число исполнителей с задачами в работе. Ниже: столбчатый график скорости по последним 10 завершённым спринтам, линия сгорания активного спринта вместе с идеальной (без активного спринта — пояснение вместо графика), накопительная диаграмма потока за 30 дней с областью на каждый статус и таблица загрузки по людям. Окна (10 спринтов, 90 дней, 30 дней) зашиты в компоненте — переключателя периода нет.

Графики рисует `components/pm/PmChart.vue` — тонкая обёртка над chart.js: библиотека подгружается динамически при первом отображении, экземпляр пересобирается при смене `type`/`data`/`options` и уничтожается на размонтировании.

### Sidebar

Collapsible секция "ЗАДАЧИ" с подпунктами: Доска, Список, Бэклог, Спринт, Гант, Календарь, Метрики. Каждый пункт — `router-link` с `?tab=` query параметром.

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

У вкладки «Метрики» горячей клавиши нет — только сайдбар и TabMenu.

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

### Backend (16 файлов)

| Файл | Строк | Описание |
|------|-------|----------|
| `schema.js` | ~150 | DDL 11 таблиц + индексы |
| `service.js` | ~610 | Issues CRUD, hierarchy, history, каскад мягкого удаления, bulk в транзакции, чек-лист, перенос между досками |
| `views.js` | ~90 | Настройки вида: разрешение (личное > дефолт > встроенное), upsert личного, дефолт доски |
| `sprints.js` | ~160 | Sprint lifecycle |
| `csv.js` | ~45 | CSV-экспорт задач |
| `board.js` | ~55 | Board/backlog aggregation |
| `boards.js` | ~220 | Доски: перечень, создание с посевом статусов, удаление, CRUD статусов доски |
| `statuses.js` | ~35 | Перечень статусов (по доске/весь), доску-осведомлённый statusJoin |
| `comments.js` | ~65 | Comments CRUD |
| `links.js` | ~35 | Issue-to-issue links |
| `watchers.js` | ~30 | Watch/unwatch |
| `data-links.js` | ~40 | Data links |
| `templates.js` | ~45 | Issue templates |
| `metrics.js` | ~160 | 6 metric functions |
| `listeners.js` | ~165 | PR auto-link, notifications, automations |
| `router.js` | ~385 | HTTP endpoints |
| `recurring.js` | ~100 | Cron job для повторяющихся задач |

### Frontend

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
| `views/pm/MetricsView.vue` | Экран метрик |
| `composables/usePmShortcuts.js` | Горячие клавиши PM |
| `components/pm/PmChart.vue` | Обёртка над chart.js |
| `components/pm/IssueCard.vue` | Карточка |
| `components/pm/IssueDetail.vue` | Детали |
| `components/pm/IssueForm.vue` | Форма создания |
| `components/pm/IssueChecklist.vue` | Чек-лист |
| `components/pm/RecurrencePicker.vue` | Повторение |
| `components/pm/StatusBadge.vue` | Бейдж статуса |
| `components/pm/PriorityBadge.vue` | Бейдж приоритета |
| `components/pm/SprintHeader.vue` | Заголовок спринта |

### AI Tools (в `ai/agent/tools/pm.js`)

56 tool handler функций, TOOL_DEFS в `agent/index.js`, cases в `tool-executor.js`, тиры в `risk-tiers.js`. Полное покрытие HTTP API модуля.

### Тесты

```
backend/src/api/v2/modules/pm/__tests__/          — schema, service, sprints, board,
  sub-services, integration, listeners, metrics, cfd, create-issue, list-issues,
  export-csv, recurring, sprint-dates, router-*, а также bulk-transaction,
  reparent, soft-delete-cascade, checklist-toggle, complete-sprint

backend/src/api/v2/modules/orgs/__tests__/        — pm-aggregation, pm-integration,
  delete-org, get-org, has-pm-tables, org-wide-access, role-level,
  router-workspaces, create-issue-in-workspace

backend/src/api/v2/modules/ai/agent/__tests__/    — pm-tools

backend/src/utils/__tests__/with-transaction.test.js

frontend/src/views/pm/__tests__/                  — PmLayout, MetricsView
frontend/src/components/pm/__tests__/             — IssueChecklist, PmChart
frontend/src/services/__tests__/pm.test.js
```

Прогон: `cd backend && npx vitest run src/api/v2/modules/pm src/api/v2/modules/orgs`
и `cd frontend && npx vitest run src/views/pm src/components/pm src/services/__tests__/pm.test.js`.

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

- **Хранение**: JSONB колонка `checklist` в `_v2_pm_issues`, формат `[{id: uuid, text: string, done: boolean}]`. Признак `id` выдаётся при создании пункта и живёт вместе с ним — по нему пункт находят отметка и удаление
- **API**: три операции над одним пунктом, в ответе каждой — весь список (см. «Поведение, о котором стоит знать»):
  - `PATCH /:db/pm/issues/:id/checklist` — переключить пункт, тело `{ itemId, done }`
  - `POST /:db/pm/issues/:id/checklist` — дописать пункт в конец (`||`)
  - `DELETE /:db/pm/issues/:id/checklist/:itemId` — убрать пункт по его признаку
- **UI**: секция "Чек-лист (N/M)" в IssueDetail с чекбоксами, добавлением и удалением пунктов. Список следит за задачей и после ответа сервера заменяется тем, что вернул сервер, — параллельная правка иначе осталась бы невидимой
- **AI**: `pm_toggle_checklist` отмечает пункт по `item_id` (без `done` — переворачивает нынешнее состояние). `pm_update_issue` поле `checklist` не пробрасывает, а `PATCH /issues/:id` и `PATCH /issues/bulk` отвергают его с 400; отдельных инструментов на добавление и удаление пункта нет
- **Повторяющиеся задачи**: при создании копии чек-лист сбрасывается (все `done` → `false`)

---

## Повторяющиеся задачи

Автоматическое создание копий задач по расписанию.

- **Типы**: `daily`, `weekly`, `biweekly`, `monthly`
- **Хранение**: колонка `recurrence` (VARCHAR) в `_v2_pm_issues`
- **Логика**: cron job (`recurring.js`) каждый час проверяет все воркспейсы
- **Триггер**: задача с `recurrence` перешла в `done`/`canceled` → создаётся новая копия в `backlog`
- **Защита от дублей**: `NOT EXISTS` проверяет нет ли открытой копии с тем же title+recurrence
- **Чек-лист**: при создании копии все пункты сбрасываются в `done: false` и получают НОВЫЕ признаки — скопированный `id` жил бы сразу в двух задачах
- **UI**: dropdown "Повторение" в форме создания задачи (Не повторяется / Ежедневно / Еженедельно / Раз в 2 недели / Ежемесячно)

---

## Связи с данными

Задача привязывается к данным воркспейса: строка в таблице `_v2_pm_data_links` хранит пару `target_type` + `target_id`. Связь ведёт на запись/таблицу целиком, не на ячейку.

### Типы целей

| `target_type` | Цель |
|---------------|------|
| `table` | EAV-таблица (тип воркспейса) |
| `object` | Запись EAV-таблицы (существование проверяется при привязке — несуществующий id даёт 404) |
| `document` | Документ |
| `report` | Отчёт |

Неизвестный тип — 400. Дубль той же пары «задача + цель» — 409 (`UNIQUE(issue_id, target_type, target_id)`).

### Маршруты

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/issues/:id/data-links` | Привязки задачи: `{ items: [{ id, issue_id, target_type, target_id, created_by, created_at }] }` |
| POST | `/issues/:id/data-links` | Создать привязку, тело `{ targetType, targetId }` — только editor и выше (canWrite + CSRF) |
| DELETE | `/data-links/:linkId` | Удалить привязку — только editor и выше (canWrite + CSRF) |
| GET | `/issues/by-target?target_type=&target_id=` | Обратный поиск — задачи цели с номером, названием, статусом, приоритетом, исполнителем; удалённые задачи не попадают. Маршрут объявлен выше `GET /issues/:id`, иначе Express принял бы `by-target` за `:id` |

### UI

- **Панель задачи** (`IssueDetail.vue`) — блок «Связи с данными»: перечень привязок с удалением по крестику и форма привязки (плюсик). Тип «Запись таблицы» предлагает сначала таблицу, затем записи выбранной таблицы списком; тип «Таблица» — выбор таблицы. Типы «Документ» и «Отчёт» есть в селекторе, но поля ввода цели для них в форме v1 нет — привязка остаётся недоступной (кнопка «Привязать» погашена), цели этих типов заводятся через API/AI-инструмент.
- **Запись таблицы** (`views/data/ObjectView.vue`) — блок «Задачи»: связанные с записью задачи (обратный поиск по `object`), переход в раздел PM по клику, привязка задач поиском по номеру или названию. Блок не показывается, когда задач нет.

---

## Поведение, о котором стоит знать

- `PATCH /issues/bulk` и `DELETE /issues/bulk` идут в одной транзакции: сбой на
  середине откатывает весь список, а не оставляет часть правок применённой.
  События (`pm.issue.updated`, `pm.issue.status_changed`, `pm.issue.deleted`)
  копятся и уходят в шину ПОСЛЕ коммита: подписчик читает с другого соединения и
  до коммита видел бы прежние значения. Упавший подписчик пишется в лог и не
  мешает остальным — правки к тому моменту уже зафиксированы.
- `DELETE /issues/bulk` пропускает задачи, уже погашенные каскадом другой задачи
  из того же списка: иначе второй вызов ответил бы 404 и откатил всю транзакцию —
  список `[родитель, подзадача]` не удалял бы ничего, а `[подзадача, родитель]`
  отрабатывал. В ответе `deleted` — число СНЯТЫХ задач вместе с потомками, а не
  число сделанных вызовов; оно может быть и больше, и меньше длины `ids`.
- Смена `parent_id` через `PATCH /issues/:id` проверяется ДО записи. Каждый
  случай — 400:
  - задача сама себе родитель;
  - новый родитель — её потомок (цикл);
  - поддерево не влезает в `MAX_DEPTH = 3` — складывается глубина нового
    родителя и высота переносимого поддерева, а не одна задача;
  - родителя не существует или он в корзине (внешний ключ мягкого удаления не
    видит — без проверки задача повисла бы под удалённой);
  - идентификатор родителя не целое положительное число (иначе Postgres ответил
    бы `22P02` и наружу ушло бы 500 вместо 400).
- Эпик не может иметь родителя — при правке правило держится в обе стороны: и
  назначение родителя эпику, и смена типа на `epic` у задачи с родителем.
  Проверка включается, только когда правка касается `type` или `parent_id`:
  иначе уже испорченная запись перестала бы принимать любые правки, включая
  смену статуса, и починить её было бы нечем.
- `DELETE /issues/:id` гасит задачу вместе со всем поддеревом одним рекурсивным
  запросом; в ответе `deleted` — сколько записей ушло, `ids` — какие именно.
  `POST /trash/:id/restore` поднимает ровно тот же каскад — по совпадающей
  отметке `deleted_at`; подзадачи, удалённые отдельно и раньше, остаются в
  корзине, а в ответе `restored` — сколько записей вернулось. Восстановить
  подзадачу, пока её родитель в корзине, нельзя: 400 «Restore parent issue
  first» — иначе вышла бы живая задача со ссылкой на удалённую.
- Пункт чек-листа адресуется своим признаком (`id`, UUID внутри элемента JSONB),
  а не номером позиции: сосед, удаливший пункт выше по списку, сдвигал все
  остальные, и галочка ложилась на съехавший пункт — ответ 200, отмечено не то.
  `PATCH /issues/:id/checklist` принимает `{ itemId: непустая строка, done:
  boolean }` (иначе 400), `DELETE /issues/:id/checklist/:itemId` — признак в
  адресе; неизвестный признак в обоих случаях 404. Признаки выдаются при
  создании — и пункту (`POST /issues/:id/checklist`), и всему списку при
  создании задачи, и при клонировании повторяющейся (там НОВЫЕ: скопированный
  признак жил бы сразу в двух задачах). Старым спискам признаки проставлены
  разово при создании таблиц.
- Поле `checklist` в `PATCH /issues/:id` и `PATCH /issues/bulk` отвергается с
  400: массив снимается при открытии карточки и записывается обратно уже
  устаревшим — правка соседа откатывалась молча. Пункты правятся по одному.
- Все три операции над пунктом пересобирают список ОДНИМ оператором из нынешнего
  значения строки, а не «прочитать → поправить → записать целиком»: правка
  соседа, попавшая в базу между чтением и записью, не теряется. Добавление —
  конкатенация `COALESCE(checklist, '[]') || ?::jsonb` (пустой `text` — 400,
  несуществующая задача — 404); отметка и удаление — пересборка через
  `jsonb_array_elements(...) WITH ORDINALITY` с поиском по `->>'id'`, а 404
  держится на `EXISTS` в `WHERE`. Удаление последнего пункта оставляет `[]`, а не
  `NULL` (`COALESCE` вокруг пересборки). Каждая отдаёт весь список после правки и
  шлёт `pm.issue.updated`: прямая правка в JSONB идёт мимо `updateIssue`, и без
  явного вызова вебхуки с автоматизациями замолчали бы молча.
- Смена `parent_id` идёт под консультативной блокировкой на рабочую область
  (`pg_advisory_xact_lock(hashtext('pm_reparent:<db>'))`) внутри транзакции:
  проверка читает цепочку предков, решает и пишет, а два одновременных перевеса
  «A под B» и «B под A» проходили каждый по своему снимку и вместе создавали
  цикл. Блокировка не построчная намеренно — цепочки предков у таких перевесов
  идут в разные стороны, и `FOR UPDATE` дал бы взаимную блокировку. Транзакция
  открывается только ради этого: правка, не касающаяся родителя, идёт как
  раньше. `PATCH /issues/bulk` берёт ту же блокировку, когда в полях есть
  родитель; повторный захват внутри одной транзакции проходит, поэтому массовый
  путь себя не блокирует.
- `POST /trash/:id/restore` шлёт `pm.issue.updated` на КАЖДУЮ поднятую задачу
  (`fields: { deleted_at: null }`, с автором правки — и из маршрута, и из
  инструмента ИИ). Прежде восстановление молчало: поднятое поддерево не
  появлялось на чужих досках до перезагрузки, а кэш сводок организации оставался
  старым. Сброс этого кэша свёрнут по рабочей области окном в 200 мс — каскад из
  N задач даёт один обход и один сброс, а не N.
- `POST /sprints/:id/complete` возвращает `stats: { done, remaining, velocity, rolledOver }` —
  та же сводка, что уходит в событие `pm.sprint.completed`.
