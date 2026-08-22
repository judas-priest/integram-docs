# Integram Backend — документация

## Содержание

- [Архитектура](#архитектура)
- [Запуск](#запуск)
- [Модули](#модули)
- [Middleware](#middleware)
- [Утилиты](#утилиты)
- [Event Bus](#event-bus)
- [WebSocket](#websocket)
- [BullMQ очереди](#bullmq-очереди)
- [База данных](#база-данных)
- [AI-агент](#ai-агент)
- [OpenAPI](#openapi)
- [Тесты](#тесты)
- [Скрипты](#скрипты)

---

## Архитектура

```
                         ┌─────────────┐    ┌─────────────┐
                         │   Frontend   │    │   Portal    │
                         │  Vue 3 :5173 │    │ Nuxt 3 :3000│
                         └──────┬───────┘    └──────┬──────┘
                                │ HTTP / WS         │ HTTP
                                ▼                   ▼
┌──────────────────────────────────────────────────────────────┐
│                     Express.js :8081                           │
│                                                               │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌────────────┐  │
│  │Middleware │  │  Router   │  │  Service   │  │   Utils    │  │
│  │ jwt-auth  │  │ /api/v2/  │  │  (логика)  │  │ event-bus  │  │
│  │ db        │  │ :db/...   │  │            │  │ queue      │  │
│  │ csrf      │  │           │  │            │  │ grants     │  │
│  │ validate  │  │           │  │            │  │ cache      │  │
│  └──────────┘  └──────────┘  └───────────┘  └────────────┘  │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐     │
│  │                    Event Bus                         │     │
│  │  object.* → audit, webhooks, automations, graph,     │     │
│  │             embeddings, ws-broadcast, behavioral      │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────┐   ���
│  │ WebSocket │  │ BullMQ   │  │ Embedding│  │ AI Agent  │   │
│  │ (ws.js)  │  │ Workers  │  │ Worker   │  │ 24 agents │   │
│  └──────────┘  └──────────┘  └──────────┘  │ 530+tools │   │
│                                             └───────────┘   │
└──────────┬────────────────────┬──────────────────────────────┘
           │                    │
     ┌─────▼──────┐      ┌─────▼─────┐
     │ PostgreSQL  │      │  Valkey   │
     │ (schemas)   │      │           │
     │ EAV + _v2_* │      │ очередь   │
     │ pgvector    │      │ pub/sub   ���
     └────────────┘      └───────────┘
```

### Multi-tenancy

Каждый workspace = отдельная **схема** PostgreSQL. По умолчанию все схемы в одной БД `integram`. Параметр `:db` в URL определяет схему. Глобальные таблицы (`_v2_users`, `_v2_workspaces`, `_v2_portal_config`) живут в public-схеме основного сервера.

**Remote workspace databases:** Отдельные воркспейсы могут хранить данные на удалённом PostgreSQL-сервере. Колонка `_v2_workspaces.remote_dsn` (JSONB) содержит `{host, port, database, user, password}`. При старте `loadRemoteDsns()` регистрирует DSN в `pool-manager.js`, который кеширует `pg.Pool` по connection string. `dbMiddleware` вызывает `getPoolForDb(dbName)` и ставит `req.pool` на нужный пул. Для воркспейсов без `remote_dsn` возвращается дефолтный singleton — поведение идентично прежнему. Workers и WebSocket тоже резолвят пул через `getPoolForDb(db)`. Глобальные запросы (IAM, graph, agent_memory) используют `getPool()` — всегда основной сервер.

### Ключевые паттерны

- **Event Bus** — модули общаются через события, не через прямые импорты
- **BullMQ** — тяжёлые задачи уходят в очередь (Valkey), fallback на in-process если Valkey недоступен
- **Grants** — права на уровне типа + row-level rules (OWNER_ONLY, FILTER, DENY_ALL)
- **Computed columns** — LOOKUP, ROLLUP, FORMULA вычисляются при чтении
- **sideEffect()** — fire-and-forget обёртка для фоновых операций
- **Lazy side tables** — per-workspace таблицы создаются при первом обращении (`registry/lazy-init.js`)

---

## Запуск

### Entry points

| Скрипт | Назначение |
|--------|------------|
| `scripts/start.js` | Production — полный стек: Express, WebSocket, Git Smart HTTP, HTTPS, PM2 |
| `scripts/start-pg.js` | Dev — только Express + WebSocket + SPA catch-all |
| `scripts/start-worker.js` | **BullMQ-воркеры нормалайзера** — отдельный процесс без `--watch`. Запускается `dev-start.sh` и живёт независимо от HTTP-сервера. Логи: `/tmp/nous-worker.log` |

### Порядок инициализации (`src/api/v2/index.js`)

1. Глобальный middleware: cookieParser, requestId, jwtAuth (optional), rateLimit
2. DDL: registry tables, graph schema, forms/dashboards/portal tables, load remote DSNs
3. Workers: webhooks, autofill, AI column, automations, embeddings, doc-processor (**normalizer workers — в отдельном процессе `start-worker.js`**)
4. Memory: swarm-memory init, KAG init, graph backfill
5. Event bus listeners: graph, audit, embedding, webhooks, automations, comments, documents, ws, behavioral, flat-views, AI column
6. Route registration: global → public → workspace-scoped

---

## Модули

Все модули в `src/api/v2/modules/`. Каждый содержит `router.js` (Express-роутер) и `service.js` (бизнес-логика).

**Подробная документация по каждому модулю:** [docs/modules/README.md](modules/README.md)

### Ядро

| Модуль | Описание |
|--------|----------|
| **objects** | CRUD для EAV-объектов. Пагинация, фильтрация, сортировка, bulk-операции, CSV-импорт, корзина, история изменений. Автоматические поля (created_by, timestamps). Row-level права, computed columns |
| **schema** | DDL: типы (таблицы) и реквизиты (колонки). Типы данных, модификаторы. Flat views, schema audit, snapshots, computed columns |
| **views** | Сохранённые представления: фильтры, видимые колонки, сортировка, группировка. Публичный шаринг по токенам |
| **lookups** | Провайдеры данных для выпадающих списков |

### AI

| Модуль | Описание |
|--------|----------|
| **ai** | Крупнейший модуль. LLM-прокси, 24 агента, 530+ инструментов, автозаполнение, AI-колонки, NL→SQL, HITL, quality worker, background scans. Подробнее: [AI-агент](#ai-агент) |
| **swarm-memory** | Персистентная AI-память. Факты, рецепты, антипаттерны. Hybrid search (vector + FTS + recency + importance). KAG. Behavioral collector. Подробнее: [docs/archive/swarm-memory.md](../../docs/archive/swarm-memory.md) |

### Документы и файлы

| Модуль | Описание |
|--------|----------|
| **documents** | Блочный редактор (Quill delta). Папки, теги, версии, шаринг, delta-синхронизация через WebSocket. Экспорт PDF/DOCX (Carbone) |
| **files** | Загрузка/скачивание (до 50 MB). PDF/DOCX парсинг, OCR, автоиндексация |

### Командный чат и коллаборация

| Модуль | Описание |
|--------|----------|
| **teamchat** | Командный чат с обязательными топиками. Комнаты, топики (M:N junction), сообщения (hybrid FTS+vector search), участники, WebSocket real-time. AI-агенты как участники. Консолидация контекста. Дебаты между агентами. Approval flow. |
| **decisions** | Граф инженерных решений. CRUD + связи (compatible/conflicts/parent/related). Авто-анамнезис (векторный + графовый + KAG). Интеграция с teamchat (чат-комната на решение). |

### Автоматизации и интеграции

| Модуль | Описание |
|--------|----------|
| **automations** | 12 типов триггеров: on_create, on_update, on_delete, schedule, manual, on_deadline, on_webhook, on_form_submit, on_document, ai_analysis, on_metric_threshold, on_metric_silence. BullMQ с retry |
| **webhooks** | Исходящие HTTP-уведомления на CRUD-события. HMAC-подпись, retry, dead letter queue |
| **connectors** | Интеграции с внешними API. AI-генерация конфигов. Пресеты. CDEK-коннектор. Инкрементальный режим stop-when-seen |
| **notifications** | In-app уведомления. Прочитано/не прочитано |
| **comments** | Треды комментариев на объектах с реакциями |

### Отчёты и аналитика

| Модуль | Описание |
|--------|----------|
| **reports** | SQL-отчёты с агрегацией (SUM, COUNT, AVG, MIN, MAX), фильтрами, пагинацией, CSV-экспортом |
| **dashboards** | Конфигурируемые дашборды с виджетами (JSONB) |
| **timeseries** | Временные ряды. TimescaleDB гипертаблица (fallback на обычную). time_bucket-агрегация |

### Портал

| Модуль | Описание |
|--------|----------|
| **portal** | Клиентский портал. Telegram-аутентификация (бот + OTP SMS опционально), каталог, корзина, заказы, профиль, KB, документы, метрики, AI-чат, аналитика, роли через EAV, guest orders. Подробнее: [portal/docs/PORTAL.md](../../portal/docs/PORTAL.md) |

### Управление

| Модуль | Описание |
|--------|----------|
| **workspaces** | CRUD workspace. Клонирование, боты, приглашения, участники. Workspace templates — манифестное создание/восстановление с ремаппингом EAV-ID |
| **iam** | Глобальная аутентификация. Регистрация, логин/логаут, JWT refresh, forgot/reset password |
| **auth** | Legacy, заменён iam |
| **orgs** | Организации — группировка workspace |
| **admin** | Бэкап/восстановление, права, dead letter queue, QA bugs |
| **templates** | Record templates — предзаполненные значения полей |

### Граф и поиск

| Модуль | Описание |
|--------|----------|
| **graph** | Графовый слой на PostgreSQL (graph_objects + graph_edges). pgvector search. Обход соседей, паттерн-матчинг |
| **computed-reqs** | Редирект на schema/computed |

### Прочее

| Модуль | Описание |
|--------|----------|
| **normalizer** | AI-нормализация документов. Two-pass extraction (entities + doc blocks), workspace-aware resolver, dedup upsert. Agents: coordinator, classifier, extractor, resolver, architect, populator, doc-writer, dash-builder. Workers в отдельном процессе |
| **agent-registry** | Реестр внешних AI-агентов. Delegation, HMAC, health-check |
| **codespace** | Git-хостинг. Smart HTTP, PR, хуки. Лимит 500 MB/репо |
| **audit-export** | Единая выгрузка audit-логов |
| **forms** | Публичные формы по токену |
| **desktop** | Персистентные вкладки пользователя |
| **testing** | Внутренний QA |
| **calls** | P2P голосовые/видео звонки + комнатный голосовой чат. WebRTC через Go signal-server |
| **tts** | Text-to-speech синтез |
| **http-button** | Колонка-кнопка: HTTP-запрос из ячейки таблицы с подстановкой данных строки |
| **script-button** | Колонка-кнопка: пользовательский JavaScript в браузерном Web Worker |
| **workspace-tools** | Пользовательские JS-инструменты в V8-изоляции (isolated-vm). Per-workspace, доступны AI-агенту |
| **agent-suggestions** | HITL-подсказки от AI на основе поведенческих паттернов |
| **dlp** | Data Loss Prevention — политики защиты данных |
| **error-collector** | Сбор runtime-ошибок для мониторинга |
| **specs** | Спецификации и шаблоны инфраструктуры |

---

## Coding Patterns

### Структура модуля

- `router.js` — только HTTP-слой: разобрать запрос → вызвать сервис → отправить ответ. **Никакой бизнес-логики.**
- `service.js` — только бизнес-логика и доступ к БД. **Никаких `req`/`res`.**
- Новый модуль → `src/api/v2/modules/<name>/router.js` + `service.js`, подключить в `index.js`.

### EAV-запросы

- Имя таблицы — всегда через `eavTable(db)`, никогда хардкодом.
- Записи данных: `WHERE t = ? AND up = ?` (оба = `typeId`).
- Column definitions: `WHERE up = ? AND t != up` (`t ≠ typeId` — инвариант, отделяющий колонки от данных).
- Все запросы через `execSql(pool, sql, params)` — никогда `pool.query()` напрямую.
- SQL: только параметризованные запросы (`?`), никогда строковая конкатенация.

### Транзакции

- Несколько записей, которые должны примениться целиком или никак, — через
  `withTransaction(pool, fn)` из `utils/execSql.js`. Внутрь приходит клиент; в
  запросы передавать его, а не `pool`, иначе они уйдут мимо транзакции.
- **`withTransaction` бросает, если `COMMIT` вернул командный тег `ROLLBACK`.**
  Так Postgres отвечает на коммит аварийной транзакции — сам он при этом НЕ
  бросает, и без проверки вызывающий принимал бы потерю данных за успех.
  Касается всех вызывающих, не только PM.
- Побочные записи внутри транзакции нельзя пускать через `sideEffect`: они идут
  по тому же клиенту, их падение делает транзакцию аварийной, а сообщение о
  причине `sideEffect` проглатывает — наружу выйдет «current transaction is
  aborted» вместо настоящей ошибки. Внутри транзакции — `await`.
- События в шину — ПОСЛЕ коммита, а не внутри: подписчик читает с другого
  соединения и до коммита видит прежние значения. Накопить и разослать —
  как в `pm/service.js` (`emitAfterCommit`).

### Ошибки и события

- Всегда `throw new AppError(CODE, message, httpStatus)` — никогда `new Error()`.
- Межмодульное общение через `bus.emit('event.name', payload)`, не через прямые импорты.
- Fire-and-forget → оборачивать в `sideEffect(() => ...)`, чтобы ошибки не валили caller.

### Middleware chain (workspace-scoped routes)

`jwtAuth → dbMiddleware → workspaceRoleMiddleware` — всегда в таком порядке.

### Аутентификация

- Refresh token — **HttpOnly cookie** (невидим для JS). Access token — только в памяти.
- `/iam/refresh` читает токен из `req.cookies.refresh_token`, не из тела запроса.
- `setRefreshCookie(res, token)` / `clearRefreshCookie(res)` — хелперы в `iam/router.js`.

---

## Middleware

`src/api/v2/middleware/`

| Файл | Описание |
|------|----------|
| **jwt-auth.js** | JWT-верификация, загрузка пользователя, `workspaceRoleMiddleware` (роль в workspace + **отметка визита**, см. ниже), guards: `requireJwt`, `requireModule`, `requireRole` |
| **service-key-auth.js** | `Bearer smk_...` токены для backend-to-backend. HMAC-SHA256 + legacy SHA256 |
| **db.js** | Извлечение `:db`, резолв slug→db_name, подключение pool через `getPoolForDb()` (remote-aware). `req.pool` — workspace pool, `req.sysPool` — основной pool для глобальных таблиц, `req.db`/`req.workspace` |
| **validate.js** | Zod-валидация query/params/body → `req.validated` |
| **rate-limit.js** | In-memory sliding-window, IETF RateLimit-* headers, custom keyFn |
| **error-handler.js** | Единый формат ошибок: AppError, ZodError, SQL injection guards, 500 |
| **csrf.js** | Double-submit cookie. Exempt: Bearer/JWT/service_key, safe methods |
| **workspace-membership.js** | Проверка членства. LRU-кэш 10 мин, 1000 записей |

### Цепочка на маршрутах воркспейса

```
jwtAuth                  (глобально — кладёт req.v2user, если JWT есть)
dbMiddleware             (/:db — проверяет воркспейс, кладёт req.pool + req.db)
requireJwt               (/:db — отклоняет неаутентифицированных)
workspaceRoleMiddleware  (/:db — резолвит роль, кладёт req.user, отмечает визит)
```

Под `/:db` у `req` всегда есть `req.pool`, `req.db`, `req.user`.

**Отметка визита.** `workspaceRoleMiddleware` зовёт
`touchLastSeen(pool, workspaceId, userId)` из `modules/workspaces/last-seen.js` —
единственного писателя `_v2_memberships.last_seen_at`. Три подробности, без
которых правка ломает вход:

1. **Зовётся только на успешных ветках** (суперадмин, роль из кэша, роль из базы)
   и ни на одном из четырёх отказных путей: 403 не есть посещение.
2. **Только через `sideEffect('ws_last_seen', …)`.** Функция БРОСАЕТ — это
   объявлено в её JSDoc и противоречит соседям по коду, у которых «никогда не
   бросает» есть буквальное обещание. Голый `await` отдаёт законно разрешённый
   запрос в **403** «Workspace access denied», а не в 500: общий `catch`
   middleware принимает отказ вспомогательной записи за отказ в доступе
   (замерено подстановкой отказа в `execSql`). Сторож:
   `middleware/__tests__/last-seen-touch.test.js`.
3. **Дроссель 5 минут** — один UPDATE на пару (пользователь, воркспейс), состояние
   в памяти процесса. Без него это был бы UPDATE на каждый запрос по таблице,
   читаемой в каждом запросе.

Подробности, включая «кто отметку не получает» и почему на колонке намеренно нет
индекса, — [modules/workspaces.md](modules/workspaces.md#last-seen-last-seenjs).

---

## Утилиты

`src/api/v2/utils/`

### Ядро

| Файл | Описание |
|------|----------|
| **event-bus.js** | Singleton EventEmitter. Max 50 listeners. Auto-inject `_event`, `_ts`, `_cid` |
| **queue.js** | BullMQ: `safeGetQueue`, `registerWorker`, DLQ, cron jobs, delayed jobs. Graceful degradation без Redis |
| **respond.js** | `sendOk`, `sendCreated`, `sendError`, `sendPaginated` |
| **errors.js** | `AppError` с code, message, status, field, details |
| **cache.js** | TTL-based in-memory кэш |
| **side-effect.js** | Fire-and-forget с error logging |
| **paginate.js** | `paginate(pool, query, params, { page, pageSize })` |

### Права и безопасность

| Файл | Описание |
|------|----------|
| **v2-grants.js** | Загрузка/проверка ACL. `getUserEffectiveGrants(pool, username, roleId) → Map<typeId, {level, can_export, can_delete, mask_filter}>`, `grantsToLegacyFormat(rows) → {[typeId]: 'READ'\|'WRITE'\|'NONE', 0: ..., EXPORT: {}, DELETE: {}}`, `setGrant`, `removeGrant` |
| **row-permissions.js** | Row-level rules: OWNER_ONLY, FILTER, DENY_ALL |
| **url-guard.js** | SSRF-защита для webhook/connector URL: блокировка приватных IP, DNS pre-resolution |

### EAV и формулы

| Файл | Описание |
|------|----------|
| **computed-reqs.js** | LOOKUP, ROLLUP, FORMULA. Граф зависимостей, топологическая сортировка |
| **formula-engine.js** | Песочница для формул: math, string, date операции |
| **filter-dsl.js** | DSL → SQL WHERE: eq, neq, gt, lt, like, in и т.д. |
| **eav-col-defs.js** | Безопасный запрос column definitions |
| **column-validation.js** | Per-column validation: min/max, regex, required, unique |
| **report-engine.js** | SQL-движок отчётов: multi-join, агрегация, группировка, pivot |

### Embedding и граф

| Файл | Описание |
|------|----------|
| **embedding-sync.js** | Vector pipeline: scheduleEmbed/Delete, vectorSearchObjects/Docs, HNSW iterative scan |
| **embedding-listeners.js** | Event bus → embedding-sync |
| **chunker.js** | Разбиение текста на чанки с citation-метаданными (для RAG) |

### Аудит и трекинг

| Файл | Описание |
|------|----------|
| **audit-log.js** | `logChange`, `logDelete` — field-level audit trail |
| **audit-listeners.js** | Event bus → audit-log на object.created/updated/deleted |
| **autofields.js** | created_at/by, updated_at/by |
| **record-mentions.js** | @-упоминания в документах/комментариях, бэклинки |

### Коммуникации

| Файл | Описание |
|------|----------|
| **ws-listeners.js** | Event bus → WebSocket broadcast для objects, schema, documents, reports, notifications, workspace |
| **mailer.js** | Email через nodemailer |
| **agent-memory.js** | Per-conversation AI memory: save/recall/forget |

### Прочее

| Файл | Описание |
|------|----------|
| **bki.js** | BKI (бюро кредитных историй) интеграция |

---

## Event Bus

Singleton `EventEmitter`. Все модули общаются через события.

### События и подписчики

| Событие | Подписчики |
|---------|------------|
| `object.created` | audit-log, embedding-sync, webhooks, automations, ws-broadcast, behavioral-collector, ai-column |
| `object.updated` | audit-log, embedding-sync, webhooks, automations, ws-broadcast |
| `object.deleted` | audit-log, embedding-sync, webhooks, ws-broadcast |
| `object.requisite.changed` | audit-log |
| `object.moved` | ws-broadcast |
| `object.edge.created/deleted` | graph |
| `schema.type.created/updated/deleted` | ws-broadcast, flat-views |
| `schema.column.created/updated/deleted/reordered` | ws-broadcast, flat-views |
| `document` | embedding-sync, ws-broadcast, documents-listeners |
| `report.created/updated/deleted` | ws-broadcast |
| `notification.created` | ws-broadcast |
| `workspace.created/updated/deleted` | ws-broadcast (global) |
| `automation.saved/deleted` | automations (sync cron) |
| `comment.created` | comments-listeners |
| `grants.changed` | workspace role cache clear |

---

## WebSocket

**Файл:** `src/api/v2/ws.js`. Подробно: [modules/ws.md](modules/ws.md)

**Endpoint:** `ws://host/api/v2/:db/ws`

### Протокол

Все сообщения — JSON.

**Аутентификация:**
```json
{"type": "auth", "token": "JWT_TOKEN"}
```
Первое сообщение, таймаут 10 сек.

**Подписка:**
```json
{"type": "subscribe", "channel": "objects", "typeId": 123}
{"type": "subscribe", "channel": "documents", "workspaceId": "ws", "documentId": "abc"}
{"type": "subscribe", "channel": "sandbox-collab", "topicId": "7", "messageId": "42"}
{"type": "subscribe", "channel": "point", "pointType": "record", "pointId": 123}
```

**Каналы подписки со своей обработкой — четыре:** `objects` (по typeId; умолчание,
если `channel` не передан), `documents` (по workspaceId + documentId),
`sandbox-collab` (по topicId + messageId), `point` (по pointType + pointId;
`pointType` — только `table` или `record`, словарь `POINT_TYPES` в
`modules/presence/service.js`).

**Имена каналов в броадкасте — другой перечень.** `broadcast(db, channel, …)`
рассылает по ключу `<channel>:<typeId|*>:<parentId|*>` и не ограничен четвёркой
выше: `objects`, `documents`, `schema`, `reports`, `notifications`, `automations`,
`pm`, `files`, `ai-batch`. Плюс глобальный `broadcastAll('workspaces', …)`.

**Роды кадров клиент → сервер — двенадцать:** `auth`, `subscribe`, `unsubscribe`,
`documents`, `sandbox-collab`, `objects`, `tc:join-room`, `tc:leave-room`,
`tc:typing`, `calls`, `point`, `ping`.

**Collaborative editing:**
```json
{"type": "documents", "operation": "delta-update", "documentId": "...", "delta": {...}}
{"type": "documents", "operation": "cursor-move", ...}
{"type": "documents", "operation": "presence-ping", ...}
```

Delta persistence: debounced flush каждые 2 сек, принудительно каждые 10 сек при непрерывном редактировании.

**Rate limit:** 120 msg / 60 сек. Сжатие: perMessageDeflate (порог 1 KB).

---

## BullMQ очереди

Инфраструктура: BullMQ + Valkey. Graceful degradation без Redis — задачи выполняются in-process.

| Очередь | Worker | Concurrency | Назначение |
|---------|--------|-------------|------------|
| `automations` | automations/worker.js | 10 | Триггерные автоматизации + cron |
| `webhooks` | webhooks/worker.js | 20 | Доставка HTTP-уведомлений с retry |
| `ai-batch` | ai/autofill-worker.js | 2 | AI autofill batch |
| `ai-column` | ai/ai-column-worker.js | 5 | AI computed column |
| `background-scans` | ai/background-scans/worker.js | 2 | Фоновые AI-сканы воркспейса |
| `normalizer` | normalizer/worker.js ¹ | 3 | Координатор нормализации |
| `normalizer-classify` | normalizer/worker.js ¹ | 10 | LLM классификация документов |
| `normalizer-extract` | normalizer/worker.js ¹ | 5 | LLM извлечение данных (two-pass) |
| `normalizer-resolve` | normalizer/worker.js ¹ | 2 | LLM разрешение конфликтов |
| `normalizer-architect` | normalizer/worker.js ¹ | 2 | LLM генерация схемы |
| `normalizer-populate` | normalizer/worker.js ¹ | 2 | LLM заполнение данных |
| `normalizer-docwrite` | normalizer/worker.js ¹ | 3 | LLM генерация документов |
| `normalizer-dashbuild` | normalizer/worker.js ¹ | 2 | LLM построение дашбордов |
| `ai-quality` | 1 | 3600s | Еженедельный LLM-судья качества AI-колонок и фидбека агентов (`ai-quality-worker.js`) |

¹ Normalizer workers запускаются через `scripts/start-worker.js` — отдельный процесс без `--watch`.
| `dead-letter` | ручной retry | — | Упавшие задачи всех очередей |

Дефолт: 5 попыток, exponential backoff (5 сек база, 0.5 jitter). Хранение: 1000 completed (7 дней), 5000 failed (30 дней).

---

## База данных

### Глобальные таблицы (public schema)

`_v2_users`, `_v2_orgs`, `_v2_workspaces`, `_v2_memberships`, `_v2_invitations`, `_v2_refresh_tokens`, `_v2_password_reset_tokens`, `_v2_roles`, `_v2_desktop_tabs`, `_v2_workspace_templates`, `_v2_grants`, `_v2_service_keys`, `_v2_forms`, `_v2_dashboards`, `_v2_portal_config`, `_v2_portal_events`, `_v2_record_share_tokens`, `_v2_view_share_tokens`

### Graph/AI таблицы (public schema)

`graph_objects` (vector(1024), HNSW), `graph_edges`, `doc_chunks` (vector(1024), HNSW), `agent_memory` (vector(1024), HNSW, tsvector), `agent_procedures`, `agent_sessions`, `shared_state`, `shared_state_log`, `memory_edges`, `memory_contradictions`, `memory_audit`, `kag_entities`, `kag_classes`, `kag_relations`

### Per-workspace таблицы (lazy-init)

Создаются при первом обращении к workspace через `registry/lazy-init.js`:

`_v2_autofields`, `_v2_computed_reqs`, `_v2_audit_log`, `_v2_views`, `_v2_comments`, `_v2_webhooks`, `_v2_notifications`, `_v2_row_rules`, `_v2_automations`, `_v2_files`, `_v2_type_meta`, `_v2_schema_audit`, `_v2_schema_snapshots`, `_v2_report_audit`, `_v2_entity_meta`, `_v2_documents`, `_v2_doc_blocks`, `_v2_embedding_sync`, `_v2_column_validation`, `_v2_record_mentions`, `_v2_timeseries`

### EAV-ядро (per-workspace)

Каждый workspace содержит две основные EAV-таблицы:
- `"db"."db"` — корневая таблица объектов (id, t=typeId, up=parentId, name=_value, ord, del)
- `"db"."db"` содержит и объекты и реквизиты в одной таблице (id, t=typeId/reqTypeId, up=parentId/objectId, val=value, ord)

Индексы: `idx_up_t_ord`, `idx_t_val`, `idx_val_trgm` (GIN trigram для кириллицы)

---

## AI-агент

### Архитектура

```
Пользователь → POST /ai/agent-chat (SSE)
                    ↓
              Orchestrator
              (semantic routing)
                    ↓
         ┌─────────┼─────────┐
         ↓         ↓         ↓
    Tables Agent  Docs Agent  Reports Agent  ...
    (runner.js)   (runner.js) (runner.js)
         ↓         ↓         ↓
    tool-executor.js (switch ~530 cases + workspace tools fallback)
         ↓
    Service layer (objects, schema, documents, ...)
```

### 24 агента

| Агент | Зона ответственности |
|-------|---------------------|
| **tables** | CRUD объектов, схема, bulk, computed, history, comments |
| **reports** | Создание/редактирование отчётов, колонки, фильтры, агрегация, JOIN |
| **docs** | Документы: блоки, версии, шаринг, папки, теги, editor-команды |
| **grants** | Права доступа: роли, гранты, row-level rules |
| **automation** | Автоматизации, вебхуки, формы |
| **files** | Файлы, импорт/экспорт, коннекторы, AI-генерация коннекторов |
| **admin** | Участники workspace, настройки, бэкапы, корзина, уведомления, аудит |
| **dashboard** | Дашборды, виджеты, представления, публичный шаринг |
| **portal** | Конфигурация портала, публикация, заказы, метрики, Telegram-боты |
| **normalizer** | Pipeline нормализации документов: запуск, статус, подтверждение |
| **timeseries** | Запись и запрос временных рядов |
| **advisor** | Консультации по использованию платформы |
| **analyst** | Аналитик: агрегация, граф знаний, конфликты решений, W-матрица |
| **codespace** | Git-репозитории, бранчи, коммиты, PR |
| **meta-kb** | Мета-база знаний: дебаты экспертов, KAG, решения |
| **teamchat** | Командный чат: комнаты, топики, сообщения, решения |
| **telegram** | Telegram-боты: конструктор, навигация, платежи |
| **reviewer** | Code review: анализ PR, комментарии |
| **sandbox-controller** | Управление sandbox-сессиями |
| **sandbox-engineer** | Реализация кода в sandbox |
| **sandbox-opponent** | Ревью и критика кода в sandbox |
| **sandbox-researcher** | Исследование и поиск решений для sandbox |
| **activity** | Анализ активности команды: teamchat, meta-kb, decisions. Блокеры, коммитменты, итоги |
| **kag** | Knowledge-Augmented Generation: импорт, поиск, обход сущностей в графе знаний |

### Общие инструменты (доступны всем агентам через orchestrator)

`remember`, `recall`, `forget`, `share_insight`, `find_procedure`, `list_contradictions`, `resolve_contradiction`, `get_related`, `graph_query`, `kag_search`, `kag_traverse`, `kag_ask`, `list_agents`, `delegate_to_agent`, `search_tools`

### HITL (Human-in-the-Loop)

Опасные операции (delete, bulk_delete, schema changes) требуют подтверждения пользователя. Классификация по risk tiers в `risk-tiers.js`. Pending confirmation хранится в `history.js`, resume через `POST /ai/agent-resume` или `POST /ai/mcp-resume`.

### MCP интеграция

MCP-инструменты вызываются через `POST /api/v2/:db/ai/tool` с `{ name, args, skipHitl, callId }`. Используется тот же `tool-executor.js`. Каталог инструментов доступен через `GET /api/v2/:db/ai/tools`.

### AI endpoints

| Метод | Путь | Описание |
|-------|------|----------|
| POST | `/ai/agent-chat` | Мультиагентный SSE-стрим (основной чат) |
| POST | `/ai/agent-resume` | HITL resume после подтверждения |
| POST | `/ai/agent-elicit` | Resume после elicitation-формы |
| POST | `/ai/mcp-resume` | HITL resume для MCP (без CSRF) |
| POST | `/ai/tool` | Выполнить инструмент по имени (для MCP) |
| GET | `/ai/tools` | Список всех инструментов (для MCP) |
| POST | `/ai/chat` | LLM chat completion |
| POST | `/ai/fill-table` | AI-заполнение строк таблицы |
| POST | `/ai/column-agent` | Per-column AI agent |
| POST | `/ai/autofill-batch` | Batch AI autofill (sync или через BullMQ) |
| POST | `/ai/run-button` | Выполнить AI-кнопку |
| POST | `/ai/generate-formula` | Формула из NL-описания |
| POST | `/ai/suggest` | Подсказки значений |
| POST | `/ai/query` | NL → data query |
| POST | `/ai/generate-delta` | Генерация Quill Delta документа |
| GET/PUT | `/ai/column-config/:typeId/:reqId` | Конфиг AI-колонки |
| GET | `/ai/models` | Доступные модели |
| GET | `/ai/usage` | Статистика использования LLM |
| GET | `/ai/audit-log` | Лог вызовов инструментов |
| GET/POST/DELETE | `/ai/conversations[/:id]` | CRUD бесед |
| GET | `/ai/conversations/:id/messages` | История сообщений |
| POST | `/ai/feedback` | Обратная связь (thumbs up/down) |

---

## OpenAPI

- **Спецификация:** OpenAPI 3.1.0
- **JSON:** `GET /api/v2/openapi.json`
- **UI (Scalar):** `GET /api/v2/docs`
- **Файл:** `src/api/v2/openapi.js`
- **Auth:** Bearer JWT
- **43 тега:** IAM, Workspaces, Orgs, Desktop, Schema, Lookups, Objects, Objects - Bulk, Objects - History, Reports, Views, Documents, Files, AI, Graph, Swarm Memory, Automations, Webhooks, Connectors, Notifications, Comments, Forms, Templates, Admin, Search, Codespace, Agent Registry, Audit, Directories, Portal, Portal - Admin, CDEK, Normalizer, Timeseries, Testing, Dashboards, Health

---

## Тесты

**Runner:** Vitest (`.spec.js` / `.test.js`)

| Расположение | Файлов | Что тестируется |
|-------------|--------|----------------|
| `modules/ai/agent/__tests__/` | 10 | Инструменты агентов, HITL, plan_schema, elicitation, timeseries |
| `modules/ai/agent/tools/__tests__/` | 1 | list_objects where-фильтр |
| `modules/ai/__tests__/` | 3 | AI column listeners, autofill worker, chat timeout |
| `modules/swarm-memory/__tests__/` | 9 | Behavioral, recall tiers, procedures, security, context compression |
| `modules/swarm-memory/` | 1 | swarm-memory.test.js (интеграционный) |
| `modules/connectors/__tests__/` | 2 | AI connector tools, connector import |
| `modules/files/__tests__/` | 3 | Doc processor, file service, page texts |
| `modules/graph/__tests__/` | 1 | Graph service |
| `modules/webhooks/__tests__/` | 2 | Enqueue, worker |
| `modules/workspaces/__tests__/` | 1 | Settings normalization |
| `utils/__tests__/` | 3 | Chunker, embedding sync |

Паттерн: unit-тесты с vi.mock для изоляции DB/LLM. Интеграционные тесты — через Playwright E2E во фронтенде.

---

## Скрипты

`backend/scripts/`

### Запуск

| Файл | Описание |
|------|----------|
| `start.js` | Production сервер (полный стек) |
| `start-pg.js` | Dev PG-only сервер |
| `start-worker.js` | BullMQ normalizer workers — отдельный процесс, без `--watch` |
| `deploy-pm2.sh` | PM2 production deploy |
| `deploy-bun.sh` | Deploy с Bun runtime |
| `update-and-restart.sh` | Pull + restart |

### Миграции

| Файл | Описание |
|------|----------|
| `migrate-mysql-to-pg.js` | MySQL → PostgreSQL |
| `migrate-neo4j-to-pg.js` | Neo4j → PG graph |
| `migrate-legacy.js` | Legacy data migration |
| `migrate-users-to-v2.js` | Legacy users → _v2_users |
| `import-legacy-workspace.js` | Import из legacy MySQL workspace |
| `migrate-add-indexes.js` | Добавление индексов |
| `bootstrap-pg-schemas.js` | Инициализация PG-схем для всех workspace |

### Утилиты

| Файл | Описание |
|------|----------|
| `create-v2-user.js` | CLI: создать пользователя |
| `generate-jwt-secret.js` | Генерация JWT secret |
| `verify-env.js` | Проверка env-переменных |
| `verify-ai-providers.js` | Проверка AI API ключей |
| `seed-demo-data.js` | Seed demo workspace |
| `backfill-embeddings.js` | Batch-генерация embeddings |
| `bench-eav.js` | Бенчмарк EAV-запросов |
| `test-vector-search.js` | Тест vector search |
