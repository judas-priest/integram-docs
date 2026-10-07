# Integram — архитектура backend

## Общая схема

```
                         ┌─────────────┐
                         │   Frontend   │
                         │  (Vue 3)     │
                         └──────┬───────┘
                                │ HTTP / WebSocket
                                ▼
┌───────────────────────────────────────────────────────────┐
│                     Express.js API                         │
│                                                            │
│  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌───────────┐  │
│  │Middleware│  │  Router   │  │  Service  │  │  Utils    │  │
│  │ jwt-auth │  │ /api/v2/  │  │  (логика) │  │ event-bus │  │
│  │ db       │  │ :db/...   │  │           │  │ queue     │  │
│  │ grants   │  │           │  │           │  │ grants    │  │
│  │ validate │  │           │  │           │  │ cache     │  │
│  └─────────┘  └──────────┘  └──────────┘  └───────────┘  │
│                                                            │
│  ┌────────────────────────────────────────────────────┐   │
│  │                    Event Bus                        │   │
│  │  object.created → webhooks, automations, graph,     │   │
│  │                    embeddings, audit, ws-broadcast   │   │
│  └────────────────────────────────────────────────────┘   │
│                                                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                │
│  │ WebSocket │  │ BullMQ   │  │ Embedding│                │
│  │ (ws.js)  │  │ Worker   │  │ Worker   │                │
│  └──────────┘  └──────────┘  └──────────┘                │
└──────────┬──────────┬───────────────┬─────────────────────┘
           │          │               │
     ┌──────────┐  ┌─────────┐
     │PostgreSQL│  │ Valkey  │
     │ (schemas)│  │         │
     │ EAV +    │  │ очередь │
     │ _v2_*   │  │ задач   │
     │ pgvector │  │ pub/sub │
     └──────────┘  └─────────┘
```

## Multi-tenancy

Каждый workspace = отдельная **схема** в PostgreSQL (одна БД `integram`). Параметр `:db` в URL определяет к какой схеме идёт запрос. EAV-таблица workspace — `"db"."db"`, сателлитные таблицы — `"db"."_v2_*"`. Изоляция данных — на уровне PostgreSQL-схем. Глобальные таблицы (workspaces, users, memberships, portal_config) живут в публичной схеме.

> **Было MariaDB** — в процессе миграции перешли на PostgreSQL. Синтаксис SQL переписан: CAST AS INTEGER вместо UNSIGNED, STRING_AGG вместо GROUP_CONCAT, CASE WHEN вместо IF(), двойные кавычки вместо backtick-идентификаторов.

## Ключевые паттерны

- **Event Bus** — модули общаются через события, не через прямые импорты
- **BullMQ** — тяжёлые задачи уходят в очередь (Valkey), fallback на in-process если Valkey недоступен
- **Grants** — права на уровне типа + row-level rules (OWNER_ONLY, FILTER, DENY_ALL)
- **Computed columns** — LOOKUP, ROLLUP, FORMULA вычисляются при чтении, не хранятся
- **sideEffect()** — fire-and-forget обёртка для фоновых операций (логирование, броадкаст), никогда не бросает исключений в caller

---

## Модули (`backend/src/api/v2/modules/`)

### objects
Ядро системы. CRUD для объектов (строк) с реквизитами (колонками). Пагинация, фильтрация, сортировка, bulk-операции, CSV-импорт, корзина, история изменений. Автоматические поля: created_by, modified_by, timestamps. Проверяет row-level права и вычисляет computed columns.

### schema
DDL-операции: создание/изменение/удаление типов (таблиц) и реквизитов (колонок). Управляет типами данных, модификаторами (:ALIAS, :!NULL, :MULTI), иконками. Все изменения пишутся в schema audit log.

### ai
LLM-прокси и AI-фичи: чат с агентом (50+ инструментов), автозаполнение таблиц, настройка AI-колонок, NL→SQL перевод, подсказки значений. Human-in-the-loop для опасных действий. Интеграция с KodaCode (OpenAI-совместимый API).

### swarm-memory
Персистентная память AI-агента. Три типа: факты (semantic), рецепты (episodic), антипаттерны (negative knowledge). Hybrid search (vector + BM25 + recency + importance). SharedState — blackboard для обмена данными между сессиями. Audit trail всех операций с памятью.

### graph
Графовый слой на PostgreSQL. Объекты синхронизируются как ноды (`graph_objects`), reference-реквизиты — как рёбра (`graph_edges`). Vector search по эмбеддингам (pgvector). Поиск кратчайшего пути, обход соседей, паттерн-матчинг. Используется для визуализации схемы и семантических запросов.

### documents
Блочный редактор документов. Папки, теги, шаблоны, шаринг, версионирование, delta-синхронизация через WebSocket. Экспорт в PDF/DOCX (Carbone). Публичный доступ по invite-токенам.

### automations
Движок автоматизаций. Триггеры: on_create, on_update, on_delete, cron, manual, on_deadline. Действия: обновить поле, создать/удалить объект, отправить уведомление, вызвать webhook, запустить коннектор. Выполнение через BullMQ с retry и exponential backoff.

### webhooks
Исходящие вебхуки на CRUD-события объектов. HMAC-подпись, async dispatch через очередь, retry при ошибках, dead letter queue.

### connectors
Интеграции с внешними API. CRUD определений коннекторов (URL, headers, body template). Подстановка параметров из контекста объекта. Вызываются из автоматизаций.

### workspaces
Управление workspace: создание, список, настройки, участники, роли, приглашения, клонирование, боты. Workspace templates — манифестное создание воркспейса по шаблону: копируются типы, реквизиты, автоматизации, порталконфиг с ремаппингом EAV-ID. Настройки определяют включённые модули (документы, автоматизации, граф, AI).

### auth
JWT-аутентификация внутри workspace. Регистрация, сброс/смена пароля, проверка состояния сессии. Логин/логаут — в модуле iam.

### iam
Глобальная аутентификация (без :db). Логин, логаут, refresh токена, регистрация, forgot/reset password, профиль пользователя.

### reports
SQL-отчёты с конфигурируемыми колонками, фильтрами, агрегацией (SUM, COUNT, AVG, MIN, MAX). Пагинация, экспорт в CSV.

### forms
Публичные формы для сбора данных без логина. Доступ по токену, срок действия, привязка к типу. Ответы сохраняются как объекты.

### notifications
Уведомления пользователей. Источники: автоматизации, упоминания в комментариях, approval-запросы. Статус прочитано/не прочитано.

### comments
Треды комментариев на объектах с реакциями/эмодзи.

### files
Загрузка/скачивание файлов (до 50 MB). Метаданные в _v2_files. Интеграция с doc-processor для автоиндексации.

### views
Сохранённые представления таблиц: фильтры, видимые колонки, сортировка, группировка. Личные и общие.

### computed-reqs
Виртуальные колонки: LOOKUP (значение из связанного объекта), ROLLUP (агрегат по связанным), FORMULA (вычисляемое выражение). Граф зависимостей с топологической сортировкой. Вычислитель один — `evalComputedReqsBatch`; одиночный `evalComputedReqs` вызывает его на карте из одной записи, поэтому и список, и карточка отдают значение под ключом `computed_<id>` и в одном виде (числовые агрегаты — числа). В ответе одиночной записи дополнительно проставлен псевдоним по имени реквизита — для скриптов песочницы и AI-инструментов.

### lookups
Провайдеры данных для выпадающих списков в UI. Поиск значений по типу или реквизиту.

### orgs
Организации — группировка workspace. Участники, роли, приглашения.

### admin
Системные операции: бэкап/восстановление БД, управление правами, dead letter queue, row-level rules.

### desktop
Персистентные вкладки пользователя. Хранит открытые таблицы/отчёты/документы с позицией и настройками.

### templates
Шаблоны записей (record templates) — предзаполненные наборы значений полей, привязанные к типу (`_v2_templates`). Пользователь сохраняет набор полей и применяет его при создании объекта. Не путать с workspace templates (в модуле workspaces).

### dashboards
Конфигурируемые дашборды с виджетами. Хранят layout и конфигурацию виджетов в JSONB (`_v2_dashboards`). Виджеты могут ссылаться на отчёты, таблицы, графики. Владелец и права на уровне workspace.

### timeseries
Универсальное хранилище временных рядов. Любой EAV-объект (машина, датчик, договор) может быть источником данных. Точки пишутся в гипертаблицу `_v2_timeseries` (TimescaleDB, fallback на обычную таблицу). Запросы с time_bucket-агрегацией, компрессия после 2 дней, список источников с последним значением.

### normalizer
AI-нормализация документов из папки. Запускает pipeline: классификация файлов → извлечение данных → заполнение EAV-объектов → HITL-подтверждение. Состояние job хранится в swarm-memory. Агенты (`agents/`): coordinator, classifier, extractor. Выполнение через BullMQ.

### portal
Клиентский портал поверх workspace. Публичный сайт с каталогом, корзиной, заказами, личным кабинетом, юридическими страницами. Аутентификация через OTP (SMS). Конфиг хранится в `_v2_portal_config` (глобальная таблица). Кастомные домены с верификацией. Proxy к Nuxt SSR (порт 3000) для SSR-страниц. Роли клиентов через EAV-таблицы workspace. Встроенный чат. Аналитика событий. Интеграция с СДЭК для расчёта доставки.

### agent-registry
Реестр внешних AI-агентов. Регистрация сторонних агентов (URL, capabilities, utterances, auth). Делегирование задач через sync/async callback. Capabilities пишутся в swarm-memory для семантического роутинга. HMAC-подпись запросов. Мониторинг health-status агентов. Таблицы: `_v2_agent_registry`, `_v2_agent_tasks`.

### codespace
Git-хостинг внутри workspace. Репозитории хранятся на диске (`GIT_ROOT`), метаданные — в `_v2_git_repos`. Pull requests с комментариями (`_v2_git_pull_requests`, `_v2_git_pr_comments`). Smart HTTP git-сервер (push/pull по HTTP). Лимит размера репозитория 500 MB. Git-хуки для броадкаста событий через event bus.

### audit-export
Единая точка выгрузки всех audit-логов workspace. Объединяет `_v2_audit_log` (объекты), `_v2_schema_audit` (схема), `_v2_report_audit` (отчёты). Фильтрация по типу, актору, дате, действию. Только для admin.

### testing
Внутренний модуль QA-тестирования. Сессии тестирования с результатами (pass/fail/skip) по test plan. Таблицы `qa_test_sessions` / `qa_test_results` в глобальной схеме. Используется командой разработки для фиксации результатов ручного тестирования.

### teamchat
Внутренний мессенджер. Комнаты, топики, сообщения, решения (decisions). AI-sandbox (code generation). W-матрица (ONA) для анализа коммуникаций. Таблицы: `_v2_tc_rooms`, `_v2_tc_topics`, `_v2_tc_messages`.

### decisions
Архитектурные решения с типизированными связями (supersedes, depends_on, conflicts_with). Векторный поиск похожих решений. Интегрирован с teamchat и meta-kb.

### meta-kb
Дебатная система курирования знаний. Параллельные эксперты + LLM-синтез. Создаёт верифицированные знания из дискуссий.

### pm
Управление проектами. Задачи (issues), спринты, доска (Kanban), бэклог. Метрики: velocity, burndown, cycle time, workload. Привязка к организациям для кросс-workspace агрегации.

### calls
P2P голосовые/видео звонки + комнатный голосовой чат. WebRTC через Go signal-server.

### presence
Журнал «кто был на рабочей точке (таблице или записи) и что там делал»: `_v2_presence_log`, срок хранения 90 дней, наполняется WS-каналом `point` и `POST /:db/presence`.

Чем НЕ является: «последний визит участника» с 20.08.2026 живёт в колонке `_v2_memberships.last_seen_at` и пишется `workspaceRoleMiddleware`, а не этим модулем. Разделение — [ADR-026](adr/026-presence-and-last-seen.md).

### tts
Text-to-speech синтез. Генерация аудио из текста.

### workspace-tools
Пользовательские инструменты (JavaScript), исполняемые в V8-изоляции (isolated-vm). Регистрируются per-workspace, доступны AI-агенту. Capabilities определяют уровень доступа.

### http-button
Колонка-кнопка: прямой HTTP-запрос из ячейки таблицы с подстановкой данных строки.

### script-button
Колонка-кнопка: пользовательский JavaScript в браузерном Web Worker.

### agent-suggestions
HITL-подсказки от AI на основе поведенческих паттернов. Предлагает автоматизации и улучшения.

### resolution
Часть normalizer pipeline — разрешение дубликатов и golden record matching.

### dlp
Data Loss Prevention — политики защиты от утечки данных.

### error-collector
Сбор и агрегация ошибок runtime для мониторинга.

### specs
Спецификации и шаблоны для генерации инфраструктуры.

### entity-meta.js
Трекинг created_by/updated_by для legacy-сущностей (отчёты, коннекторы, формы). Хранит метаданные в _v2_entity_meta.

---

## Middleware (`backend/src/api/v2/middleware/`)

| Middleware | Назначение |
|-----------|------------|
| **jwt-auth** | JWT-верификация, загрузка пользователя, роль в workspace |
| **service-key-auth** | API-ключи для backend-to-backend вызовов (smk_ токены) |
| **db** | Извлечение :db из URL, подключение к нужной схеме |
| **validate** | Zod-валидация входных данных |
| **rate-limit** | Rate limiting (auth: 10/мин, общий: по IP) |
| **error-handler** | Единый формат ошибок |
| **csrf** | Double-submit cookie CSRF-защита для cookie-аутентификации; Bearer/JWT/service_key — exempt |
| **workspace-membership** | Проверка членства пользователя в workspace (кэш 10 мин); после requireJwt и db |

## Утилиты (`backend/src/api/v2/utils/`)

| Утилита | Назначение |
|---------|------------|
| **event-bus** | EventEmitter — развязка модулей через события |
| **queue** | BullMQ-обёртка для async-задач |
| **v2-grants** | Загрузка и проверка прав доступа |
| **row-permissions** | Row-level правила (OWNER_ONLY, FILTER, DENY_ALL) |
| **computed-reqs** | Хранение и вычисление LOOKUP/ROLLUP/FORMULA |
| **formula-engine** | Парсер и вычислитель формул |
| **filter-dsl** | Парсинг фильтров в SQL WHERE |
| **embedding-sync** | Фоновая синхронизация vector embeddings |
| **embedding-listeners** | Подписчики event bus для запуска embedding-sync |
| **audit-log** | Запись изменений объектов в `_v2_audit_log` |
| **audit-listeners** | Подписчики event bus для audit-логирования (заменяет прямые вызовы logChange) |
| **ws-listeners** | Подписчики event bus для WS-броадкаста (объекты, схема, документы, отчёты, уведомления, workspace) |
| **side-effect** | Fire-and-forget обёртка: никогда не бросает, логирует ошибки |
| **cache** | TTL+LRU in-memory кэш для схемы и lookup-данных |
| **url-guard** | SSRF-защита: блокирует приватные IP/loopback, DNS pre-resolution, allowlist через CONNECTOR_ALLOWED_HOSTS |
| **autofields** | created_at/by, updated_at/by в сателлитной таблице `_v2_autofields` |
| **column-validation** | Правила валидации по колонке: minLength, maxLength, regex, unique — `_v2_column_validation` |
| **record-mentions** | Трекинг @-упоминаний объектов в документах (`_v2_record_mentions`), бэклинки |
| **report-engine** | SQL-движок отчётов (порт с MySQL на PostgreSQL), агрегации, subquery, BKI-функции |
| **bki** | BKI export/import: MaskDelimiters/HideDelimiters/UnHideDelimiters хелперы для legacy-формата |
| **chunker** | Рекурсивное разбиение текста на чанки с citation-метаданными (для RAG) |
| **eav-col-defs** | Безопасный запрос column definitions из EAV (исключает data records по инварианту t != typeId) |
| **agent-memory** | PostgreSQL-backed key/value память для AI-агентов (таблица `agent_memory`) |
| **mailer** | Отправка email |
| **paginate** | Пагинация (limit/offset) |
| **respond** | Стандартный JSON-ответ |
| **errors** | Кастомные классы ошибок |

## WebSocket (`backend/src/api/v2/ws.js`)

- Аутентификация по JWT (первое сообщение, таймаут 10 сек)
- **Каналы подписки со своей обработкой — четыре:** `objects` (по typeId; умолчание, если `channel` не передан), `documents` (по workspaceId+documentId), `sandbox-collab` (по topicId+messageId, Yjs-синхронизация ячеек песочницы), `point` (рабочая точка — таблица или запись; «кто здесь сейчас», см. `backend/docs/modules/presence.md`)
- **Имена каналов в броадкасте — это другой перечень.** `broadcast(db, channel, …)` рассылает по ключу подписки `<channel>:<typeId|*>:<parentId|*>` и не требует, чтобы имя было среди четырёх выше: `objects`, `documents`, `schema`, `reports`, `notifications`, `automations`, `pm`, `files`, `ai-batch`. Глобальный `broadcastAll('workspaces', …)` — поверх всех воркспейсов
- Роды кадров клиент → сервер — двенадцать (`auth`, `subscribe`, `unsubscribe`, `documents`, `sandbox-collab`, `objects`, `tc:join-room`, `tc:leave-room`, `tc:typing`, `calls`, `point`, `ping`)
- Real-time: CRUD-броадкаст, delta-синхронизация документов, курсоры пользователей, schema events (create/update/delete type, add/update/delete/reorder column)
- Броадкаст управляется через **ws-listeners** — подписчики event bus, декаплированные от producer-модулей
- Rate limit: 120 сообщений / 60 сек на соединение
- Сжатие: perMessageDeflate (порог 1 KB)

## Связи между модулями

```
objects ──event──► webhooks (HTTP-уведомления)
objects ──event──► automations (триггеры)
objects ──event──► graph (синхронизация нод, PostgreSQL)
objects ──event──► embedding-sync (vector index)
objects ──event──► ws-broadcast (real-time UI)  ← через ws-listeners
objects ──event──► audit-log (история изменений) ← через audit-listeners

schema ──event──► ws-broadcast (schema events)   ← через ws-listeners

automations ──import──► notifications (send_notification)
automations ──import──► connectors (run_connector)
automations ──import──► objects (create/update/delete)

ai ──import──► objects, schema, documents, graph, swarm-memory (через tools)

agent-registry ──import──► swarm-memory (capabilities для семантического роутинга)

normalizer ──import──► swarm-memory (состояние job)
normalizer ──import──► ai/agent (pipeline agents)

documents ──ws──► frontend (delta-sync, курсоры)
documents ──event──► embedding-sync (индексация)
documents ──util──► record-mentions (трекинг @-упоминаний)

portal ──proxy──► Nuxt SSR :3000 (SSR-страницы)
portal ──import──► files, reports (данные для страниц)

workspaces ──template──► schema, automations, portal (воссоздание workspace по манифесту)

timeseries ──event──► bus (timeseries.ingested)

codespace ──event──► bus (git push/merge события)

teamchat ──import──► decisions (решения из обсуждений)
meta-kb ──import──► teamchat (дебаты → топики)
pm ──import──► objects, orgs (задачи в EAV, кросс-workspace метрики)
calls ──ws──► signal-server (Go, WebRTC signalling)
workspace-tools ──import──► isolated-vm (V8 sandbox)
```
