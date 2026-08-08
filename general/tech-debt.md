# Технический долг

Проблемы, которые нужно решить, но не срочно.

---

## ~~TD-001: EAV — объекты и схема в одной таблице, разделение по соглашению~~ ЗАКРЫТ

**Закрыт 14.06.2026.** Connector fix + тест (`connector-import.spec.js:264`) + gotchas-документация. За 6 недель после патча рецидивов не было. TD-002 реализован — единый фильтр `objectWhere()` теперь используется везде.

---

## ~~TD-002: MCP и REST используют разные запросы для list_objects~~ ЗАКРЫТ

**Закрыт 14.06.2026.** Реализована shared-функция `objectWhere(safeDb, alias)` в `backend/src/shared/sql-guards.js`. Заменены 34 копипастных WHERE-фильтра в 11 файлах. MCP и REST теперь используют идентичный фильтр. Задеплоено на прод, проверено — 2429 заказов отображаются корректно.

---

## ~~TD-003: script_button (type 1020) — добавить CRUD API в браузерный sandbox~~ ЗАКРЫТ

**Закрыт 14.06.2026.** Won't do without demand. Серверный `run_script` (isolated-vm) покрывает потребности CRUD. Route `/run` возвращает 410 Gone. За 6 месяцев ни одного запроса на CRUD в browser sandbox.

---

## ~~TD-004: Report engine — нет встроенной группировки дат (date grouping)~~ ЗАКРЫТ

**Закрыт 14.06.2026.** Formula workaround через `DATE_TRUNC` покрывает потребность. `DATE_TRUNC` в `ALLOWED_FUNC`. За 6 месяцев ~4 отчёта с группировкой дат. Переоткрыть если станет частой потребностью.

---

## ~~TD-005: Семантический поиск доменов в decisions~~ ЗАКРЫТ

**Закрыт 20.06.2026.** Добавлен `toLowerCase()` при записи (createDecision, updateDecision) и чтении (listDecisions). Все 7 SQL-мест с `domain = ?` покрыты через нормализацию данных на входе. Optional chaining добавлен для защиты от undefined.

---

## TD-006: Meta KB — дедупликация/объединение топиков по смыслу

**Файлы:**
- `backend/src/api/v2/modules/teamchat/service.js` — `createTopic`, `listTopics`
- `backend/src/api/v2/modules/meta-kb/service.js` — дебаты создают топики

**Проблема:**
UNIQUE на `topic.name` **снят** (Phase 8, `DROP CONSTRAINT IF EXISTS _v2_tc_topics_name_key`). `createTopic` — голый INSERT без проверки на дубликаты. Дубликаты имён разрешены, семантически близкие топики создаются как отдельные записи.

**Решение:**
`pg_trgm` (уже включен в проекте). Триграмное сходство имён ловит 90% дубликатов.

**Приоритет:** Средний — пока топиков мало. Делать когда >50 топиков.

---

## ~~TD-007: Неподключённый код (dead code audit, 11.06.2026)~~ ЗАКРЫТ

**Закрыт 20.06.2026 (категория A).** Удалены 5 файлов с 0 импортов: PortalConfigPage.vue, AppMenuItem.vue, AppFooter.vue, Users.vue, chat-tables.js.

**Закрыт 21.06.2026 (категория B).** Удалены 5 файлов незавершённых фич с 0 импортов: NormalizerWidget.vue, SimilarSuggestionHint.vue, TestingRedirect.vue, templateLibrary.js, ai-quality-worker.js.

**Оставлен намеренно:** `automations/service.js` — `stopScheduler()` (graceful shutdown, 0 вызовов, штатный паттерн).

---

## ~~TD-008: Шаблон workspace «Архипелаг» — неполный (issue #22)~~ ЗАКРЫТ

**Закрыт 20.06.2026.** Won't do — 6+ месяцев без запросов, заблокирован отсутствием требований.

---

## ~~TD-009: Среда тестбэдов — скоуп не определён (issue #24)~~ ЗАКРЫТ

**Закрыт 14.06.2026.** Спека `notebook-sandbox-design.md` помечена SUPERSEDED — заменена на `collaborative-sandbox.md`. Директория `testbed/` не существует. Старый скоуп полностью устарел.

---

## ~~TD-010: Базовые типы — сидятся 7 из 16~~ ЗАКРЫТ

**Закрыт 20.06.2026.** Works as intended — недостающие типы системные (GRANT, PWD и т.д.), добавление создаст невыбираемые строки в UI. Базовые типы работают по integer reference.

---

## ~~TD-011: Object-layer — захардкожен под Product, AI tools не подключены~~ ЧАСТИЧНО ЗАКРЫТ

**П.2 закрыт 20.06.2026.** Зарегистрированы 3 object-layer tools: `resolve_aliases` (TIER_MEDIUM), `resolve_identity` (TIER_MEDIUM), `get_canonical_movement` (TIER_LOW). Добавлены в tool-executor, risk-tiers, TOOL_DEFS.

**П.4 закрыт 20.06.2026.** Фоллбэки на 'product'/'productalias' — параметризованные defaults, не hardcode. Все 3 функции принимают aliasTypeName/canonicalTypeName.

### Остаётся

1. **`normalizeKey()` содержит `if (field === 'gtin')`** (object-layer-upsert.js:20) — единственный реальный hardcode. Не мешает до появления 2-го канонического типа.
3. **Коннекторы не знают про канонические типы** — провенанс не записывается при импорте. Делать при наличии use case.

---

## ~~TD-012a: Workspace Rules — нет явных бизнес-правил для агентов~~ ЗАКРЫТ

**Закрыт 20.06.2026.** Реализовано:
- `recallRules(db)` в swarm-memory/service.js — SELECT по тегу `workspace_rule`, agent_id `__rules__`
- Инжекция в runner.js: rules → memory → guardrails (rules первый в dynamicParts)
- 3 AI tools: `add_rule` (TIER_MEDIUM, макс 500 символов, лимит 50), `list_rules` (TIER_LOW), `remove_rule` (TIER_HIGH)
- TOOL_DEFS в index.js
- Хранение: `agent_memory` таблица, без миграций

---

## ~~TD-013: Generation guard для load-функций с race condition~~ ЗАКРЫТ

**DocumentSharing закрыт 20.06.2026.** Добавлен generation guard в `loadAll()`: `let _gen = 0`, проверка после await, guard в catch и finally.

**Остальные закрыты 21.06.2026.** Добавлены generation guards в:
- RepoView `loadBranches` — `_branchGen`
- DocumentVersions `loadVersions` — `_verGen`
- WorkspaceMembers `loadInvitations` — `_invGen`

---

## TD-014: AI Chat + Meta KB — ручной переключатель режимов вместо автомаршрутизации

**Шаг 1 закрыт 20.06.2026.** meta-kb-agent добавлен в AGENT_EXAMPLES semantic-router.js с 12 аналитическими примерами (не пересекаются с teamchat-agent). Auto-routing теперь работает — orchestrator.js `catalog.get()` обрабатывает meta-kb-agent.

### Остаётся — шаг 2 (frontend)

Убрать SelectButton переключатель AI/Meta KB в AiPanel.vue, единый UI. Средний приоритет.

---

## TD-015: Agent Registry — внешние агенты не могут предоставлять типизированные инструменты

**Файлы:**
- `backend/src/api/v2/modules/agent-registry/service.js` — CRUD агентов, `invoke()`
- `backend/src/api/v2/modules/ai/agent/tool-executor.js` — `TOOL_DEFS`, dispatch

**Проблема:**
Agent Registry позволяет регистрировать внешних агентов (`callback_url`, `capabilities`), но они вызываются только через `delegate_to_agent(slug, task)` — текстовая задача → текстовый результат. Агент не может предоставить типизированные tools (JSON Schema).

**Приоритет:** Низкий. Критерий: 2+ внешних агента с typed tools. Преждевременно.

---

## ~~TD-016: AI tools — create_topic не зарегистрирован (broken routing)~~ ЗАКРЫТ

**Закрыт 20.06.2026.** Semantic router маршрутизировал "создай топик" в teamchat-agent, но tool `create_topic` отсутствовал. Исправлено:
- tool-executor.js: case `create_topic` с вызовом `createTopic(pool, db, roomId, body, user)`
- risk-tiers.js: `create_topic: TIER_MEDIUM`
- agents/teamchat.js: tool definition с execute callback
- agents/meta-kb.js: добавлен в coreTools
- index.js: TOOL_DEFS entry

---

## TD-012b: AI tools — неполное покрытие REST endpoints (аудит 2026-06-19)

**Контекст:** Аудит показал, что tool-executor полностью покрыт агентами, но ~20% REST endpoints не имеют соответствующих AI tools.

### Критичные — ИСПРАВЛЕНО

~~**automations**~~ — перепроверка показала: полный CRUD (17 tools). Изначальная оценка была ошибочной.

~~**codespace — чтение файлов**~~ — **ИСПРАВЛЕНО**: добавлены `get_file_tree` и `read_blob`.

~~**teamchat — create_topic**~~ — **ИСПРАВЛЕНО** (см. TD-016).

### Важные (расширение возможностей AI)

~~**teamchat — остальное**~~ — **ИСПРАВЛЕНО 2026-06-21**: добавлен `update_topic` tool (name, status, pinned, assigned_to, priority, deadline_at).

~~**graph — запись**~~ — **ИСПРАВЛЕНО 2026-06-21**: добавлены `upsert_graph_node`, `delete_graph_node`, `upsert_graph_edge`, `delete_graph_edge` tools.

### Низкий приоритет (нишевые или UI-specific)

~~**documents**~~ — **ЗАКРЫТО 2026-07-10**: добавлены `import_document`, `purge_document`, `get_block_history`, `get_doc_version`, `get_doc_settings`, `update_doc_settings`, `create_doc_invite`, `list_doc_invites`, `revoke_doc_invite`.

~~**orgs**~~ — **ЗАКРЫТО 2026-07-10**: 8/9 endpoints покрыты (list, create, get, update, delete org + list/add/remove members). Единственный непокрытый — `update_org_member` (смена роли в орге). Некритично.

**portal** — 75 endpoints, 42+ read-tools. Непокрытое — мутации: cart (add/update/delete items), orders (create), support tickets (create), staff API. Portal-user admin CRUD не существует на бэкенде — клиенты авторизуются сами.

**swarm-memory** — внутренние операции (decay, compact, probe-negatives) запускаются автоматически по расписанию. AI tools не нужны — ручной триггер не требуется.

### Модули без tools (by design)

Не нуждаются в AI tools: **iam**, **calls**, **presence**, **desktop**, **testing**, **agent-suggestions**, **audit-export**.

---

## ~~TD-017: npm — мажорные обновления зависимостей (аудит 21.06.2026)~~ ЗАКРЫТ

### ~~Безопасно обновлять (низкий риск)~~ СДЕЛАНО 2026-06-21

| Пакет | Переход | Статус |
|-------|---------|--------|
| ~~**vue-router**~~ | 4.6.4→5.1.0 | СДЕЛАНО — drop-in, нет breaking changes |
| ~~**pinia**~~ | 2.3.1→3.0.4 | СДЕЛАНО — наш setup syntax не затронут |
| ~~**multer**~~ | 1.4.5→2.2.0 | СДЕЛАНО — CVE-2025-47935, CVE-2025-47944 закрыты |
| ~~**nodemailer**~~ | 8.0.4→9.0.1 | СДЕЛАНО — TLS validation по умолчанию, наш SMTP не затронут |
| ~~**vue**~~ | 3.5.30→3.5.38 | СДЕЛАНО — патч |

### Средний риск (делать после стабилизации)

| Пакет | Переход | Что даёт | Риск |
|-------|---------|----------|------|
| ~~**express**~~ | 4→5 | Async error handling, Brotli, strict path params, `req.query` read-only | СДЕЛАНО — query parser fix (qs) + wildcard route syntax fix |
| ~~**vite**~~ | 6→8 | **2.7x быстрее билды** (Rolldown/Rust вместо esbuild+Rollup) | СДЕЛАНО — 13.78s вместо 37s, все плагины совместимы |
| ~~**pino**~~ | 9→10 | ESM-only, modern transport API | СДЕЛАНО — drop Node 18 only |
| ~~**uuid**~~ | — | — | СДЕЛАНО — удалён, не импортировался ни в одном файле |
| ~~**marked**~~ | 15/17→18 | Security fixes, улучшения парсинга | СДЕЛАНО — tokenizer changes only, API stable |

### Не трогать (высокий риск, мало выгоды)

| Пакет | Переход | Почему не стоит |
|-------|---------|-----------------|
| **openai** | 4→6 | Два мажорных скачка. AI модуль (108 файлов) на v4 API. Полный рефакторинг |
| **nuxt** | 3→4 | Переструктурирование: `app/` directory, renamed composables, Nitro v3. Портал customer-facing |
| **isolated-vm** | 6→7 | Native addon, может не собраться. Критичен для script-button |
| **eslint** | 8→10 | Два мажорных скачка, полный переход на flat config |
| **jest** | 29→30 | Legacy — основной runner vitest |
| **puppeteer** | 24→25 | Основной — Playwright, puppeteer только для PDF |
| **pdf-parse** | 1→2 | Полный rewrite, проверить API совместимость |
| **supertest** | 6→7 | Dev dependency, низкий приоритет |

### ~~Минорные обновления (wanted — безопасно, патчи и security fixes)~~ СДЕЛАНО 2026-06-21

~~Backend: `pg`, `bullmq`, `axios`, `dotenv`, `ws`, `zod`, `carbone`, `cors`, `playwright`, `simple-git`, `prettier`, `nodemon`~~
~~Frontend: `primevue`, `axios`, `dompurify`, `mermaid`, `tailwindcss`, `date-fns`, `marked`, `vite`~~
~~Portal: `vue`, `isomorphic-dompurify`, `sharp`~~ — СДЕЛАНО (nuxt 3.21.8, vue 3.5.38, isomorphic-dompurify 3.18.0)

**Закрыт 21.06.2026.** Все безопасные и средне-рисковые обновления выполнены. Осталось только высокорисковое (openai 4→6, nuxt 3→4, isolated-vm 6→7) — не трогать без необходимости.

---

## TD-018: Аудит кодовых паттернов (23.06.2026)

Системный аудит backend и frontend. Каждый тезис верифицирован по коду и проверен по best practices (OWASP, Express.js Security Guide, Smashing Magazine UX).

### ~~P1 — Высокий риск~~ ЗАКРЫТ

#### ~~1. schema/service.js — cascading DELETE без транзакции~~ ИСПРАВЛЕНО 2026-07-10

`deleteType()` и `deleteColumn(forced=true)` теперь обёрнуты в `withTransaction(pool, async (client) => { ... })`. `recursiveDelete()` получает transaction client.

#### ~~2. objects/router.js — raw req.query в aggregate/pivot без валидации~~ ИСПРАВЛЕНО 2026-07-10

Все 3 маршрута (`/aggregate`, `/pivot`, `/grouped`) теперь имеют `validate()` middleware с Zod-схемами.

### P2 — Средний риск

#### ~~3. parseInt() без NaN-проверки (11 мест)~~ ИСПРАВЛЕНО 2026-07-10

Проверенные места теперь имеют `Number.isNaN()` guards или falsy-checks после parseInt. reports/router.js, objects/router.js, files/router.js — подтверждено исправление.

#### 4. validate() есть, но читают raw req вместо req.validated

**Верифицировано:** TRUE (с уточнением для reports).

- workspaces/router.js:213,221 — wizard analyze/adapt: читают `req.body` вместо `req.validated.body` (полный обход валидации)
- reports/router.js:182,200 — body корректно через `req.validated.body`, но params через `parseInt(req.params.reportId)` вместо `req.validated.params.reportId`
- reports/router.js:224 — итерация raw `req.query` для FR_/TO_ фильтров (остальные поля через `req.validated.query`)

**Best practice:** Цель validate() middleware — создать проверенный объект. Обращение к raw `req.*` после валидации полностью обнуляет её смысл. Паттерн: `const { field } = req.validated.body`.

#### 5. SQL интерполяция вместо ? плейсхолдеров

**Верифицировано:** TRUE. Template literal interpolation в objects/service.js (aggregate/pivot/grouped SQL).

Все интерполируемые значения защищены: `colId` — через parseInt (integer), `fn` — через whitelist `ALLOWED_FUNCS`, `safeDb` — через `eavTable()` sanitizer. **Неэксплуатируемо** в текущем виде, но отличается от стандартного `$N` стиля проекта. Косметический долг — исправлять при рефакторинге aggregate/pivot.

#### 6. Dashboard/frontend — silent catches на загрузках данных

**Частично исправлено.** DashboardBuilder.vue data-load catches теперь имеют `console.error`. Остаётся 1 полностью silent catch в `toggleReportColumnHidden` (DashboardBuilder.vue:194).

Также 4 silent catches в других компонентах:
- AutomationEdit.vue — 2 места (type-fields load, batch-poll error)
- GraphExplorer.vue — 2 места (KAG anomalies, non-critical catch)

**Приоритет:** Низкий. Не влияет на данные, только на отладку.

### P3 — Низкий риск

#### 7. req.body деструктуризация без || {} (~23 места)

**Верифицировано:** TRUE (9 из 9 проверенных подтверждены).

`const { field } = req.body` без optional chaining. `express.json()` гарантирует `req.body = {}` для JSON-запросов, но не для запросов без `Content-Type: application/json` или с невалидным JSON.

Основные файлы: portal/router.js (17 мест), ai/router.js (3), templates/router.js (1), meta-kb/router.js (1), files/router.js (2), graph/router.js (1), swarm-memory/router.js (1), workspaces/router.js (2), portal/chat/router.js (2).

**Best practice (Express.js docs):** "req.body is undefined by default." Хотя `express.json()` middleware обычно обеспечивает объект, defensive coding требует `req.body || {}` или Zod валидации. Реальный риск низкий при наличии middleware.

#### 8. Frontend — неопределённость формата ответа API

**Верифицировано:** TRUE (4-fallback и 3-fallback цепочки подтверждены).

Компоненты угадывают формат: `res.data || res.items || res.options || []`.

- EmbeddedObjectTable.vue:145 — 4 fallback-пути (`res` / `res.data` / `res.objects` / `res.rows`)
- ChartEmbed.vue:58 — 3 fallback-пути (`data.items` / `data.data` / `data`)
- MentionAutocomplete.vue:99,117,135 — 3 autocomplete с fallback
- IntegramReportEmbed.vue:202 — 3 поля
- MembersPage.vue:72 — `Array.isArray(res) ? res : res.data ?? []`

**Best practice:** Единый API response envelope (`{ ok, data, meta }`) + один unwrap в service layer. Multi-fallback — симптом недокументированного контракта.

#### 9. Frontend — silent .catch(() => {}) на загрузках данных

Объединён с п.6 выше. Конкретные места перечислены там.

---

## TD-019: Платформенный meta-kb — перенести фичи из портала UAV

**Дата:** 2026-07-16

Портал UAV получил две фичи, которых нет в платформенном meta-kb (`frontend/src/views/ai/AiChat.vue`):

1. **Agent toggles** — вкл/выкл экспертов перед запросом. Пользователь выбирает каких агентов подключить к дебату. Бэкенд уже готов (`enabledAgentSlugs` в ctx → `mk_start_debate` фильтрует). Нужен UI в AiChat.vue.
2. **Pulse dot + elapsed timer** — пульсирующая красная точка и таймер прошедших секунд во время ожидания ответа агента. Сейчас платформа показывает typing dots без информации о времени.

**Файлы:**
- `frontend/src/views/ai/AiChat.vue` — добавить UI для обеих фич
- `backend/src/api/v2/modules/ai/router.js` — прокинуть `enabledAgents` из body в ctx (аналогично portal/router.js)

**Приоритет:** низкий (портал — основной интерфейс для UAV)

---

## TD-020: Portal staff — поиск сотрудника по email слишком широкий

Staff login и `/api/staff/me` ищут email сотрудника через `JOIN login ON login.val = ?` — совпадение по ЛЮБОМУ полю записи, не по конкретной email-колонке. Может дать false positive если другое поле содержит такое же значение.

**Причина:** в конфиге `employeeTable` нет `emailReqId`. Есть только `typeId`, `chatIdReqId`, `roleReqId`.

**Решение:** добавить `employeeTable.emailReqId` в конфиг портала, фильтровать по конкретной колонке: `WHERE login.t = ? AND login.val = ?`.

**Файлы:**
- `backend/src/api/v2/modules/portal/router.js` — staff login (~строка 1210), `/api/staff/me` (~строка 2036)
- Валидация конфига: `router.js:1297-1302`

**Приоритет:** низкий (false positive маловероятен, консистентно с текущим кодом)

---

## TD-021: Portal — дублирование API роутов (таблицы, отчёты, объекты)

Портальный роутер (`portal/router.js`) дублирует функциональность main app API для таблиц (`/portal/api/tables/`), отчётов (`/portal/api/reports/`), объектов CRUD (`/portal/api/objects/`). Два набора роутов делают одно и то же с разной auth и разными grant-проверками.

Публичная витрина (каталог, корзина, гостевой заказ) — уникальная логика, дублирования нет. Авторизованный доступ к таблицам/отчётам — дубль main app.

**Решение (долгосрочное):** авторизованные портальные запросы к generic ресурсам (таблицы, отчёты, объекты) проксировать в main app API, резолвив auth из portal_jwt в workspace контекст. Публичная витрина остаётся как есть.

**Приоритет:** низкий (частично решено переходом на `_v2_grants` — одна grant-система, но два набора роутов остаются)

---

## TD-022: Portal — `isReportReferencedInConfig` fallback

После перехода отчётов на `_v2_grants` оставлен fallback на `isReportReferencedInConfig` для обратной совместимости с порталами, где гранты на отчёты ещё не настроены. Убрать fallback когда все порталы мигрируют.

**Файлы:**
- `backend/src/api/v2/modules/portal/router.js` — роут `/api/reports/:reportId`

**Приоритет:** низкий (убрать после полной миграции)

---

## TD-023: Report engine — расхождения с legacy PHP (аудит 2026-07-21)

**Аудит:** `docs/audits/2026-07-21-report-engine-legacy-audit.md` — 19 пунктов, из них 3 исправлены в этой сессии.

### Исправлено (2026-07-21)

1. **Legacy JOIN ON aliases** — `aj{alias}.field` / `r{id}.field` не транслировались в новые `c`/`rj` алиасы → SQL error → пустой отчёт. Добавлена трансляция + валидация.
2. **objectWhere с NOT IN subquery** — тяжёлый subquery в WHERE отчётов. Заменён на лёгкий legacy-style фильтр (`up!=0, t!=up, length(val)>0`).
3. **mainCol помечалась isMulti** — multi-detect находил `:MULTI:` на sibling-реквизите → STRING_AGG + GROUP BY схлопывали записи. Добавлен guard `!col.isMainCol`.

### Потенциальные проблемы (не горят)

**ORDER BY без числового CAST** — legacy делает `CAST(... AS DOUBLE)` для числовых колонок при сортировке, новый движок сортирует как текст ("10" < "9"). Существует с момента создания файла, никто не жаловался. Проявляется только при клике на заголовок числовой колонки.
- Файл: `backend/src/api/v2/utils/report-engine.js`, строка ~1553
- Фикс: добавить CAST для NUMBER/SIGNED base types в ORDER BY expression

**mainCol исключена из GROUP BY** — введено в `407691329` (2026-05-08) при фиксе ref+aggregate. mainCol обёрнута в MIN() вместо GROUP BY. Влияет на агрегатные отчёты с группировкой по mainCol — нетипичный кейс (обычно группируют по другим колонкам).
- Файл: `backend/src/api/v2/utils/report-engine.js`, строка ~1579
- Фикс: добавить `a.val` в GROUP BY когда mainCol видима и не агрегирована

**Нет lateral subquery wrapping** — legacy оборачивает joined таблицы в `(SELECT ... GROUP BY ...)` subquery для per-table агрегации и filter push-down. Новый движок — плоские LEFT JOINs. Риск Cartesian product при нескольких multi-valued joins. Архитектурное различие, не фиксить превентивно.

**Приоритет:** низкий — фиксить по факту обращений пользователей

## ~~Nightcall router — упрочнение авторизации~~ (2026-08-05, из code review)

Роутер `backend/src/api/v2/modules/nightcall/router.js` (весь модуль, не отдельный дифф):

1. ~~ЗАКРЫТО (2026-08-05):~~ **Нет CSRF на мутирующих роутах** — все 13 мутирующих роутов теперь запускают `csrfMiddleware` (Bearer/service-key запросы освобождены).
2. ~~ЗАКРЫТО (2026-08-05):~~ **Нет per-route ролей** — 11 мутирующих роутов требуют `editor`, `POST /proposals/:id/decide` и `POST /decisions` требуют `admin`. Тот же контур залочен на tool-пути через `ai/agent/module-gate.js` (`GOVERNANCE_TOOLS`, `GOVERNANCE_MIN_ROLE`).
3. **POST-роуты отвечают 200 вместо 201** — конвенция всего модуля; менять только вместе, иначе рассинхрон. Открыто.
- Файл: `backend/src/api/v2/modules/nightcall/router.js`
- Приоритет: низкий (остался только п.3)

## ~~Nightcall AI tools — соответствие ai-tools.md~~ — ЗАКРЫТО (2026-08-05, из code review)

1. ~~**`throw new Error(...)` вместо структурированных ошибок**~~ — все 18 argument-validation throws в `backend/src/api/v2/modules/ai/agent/tools/nightcall.js` теперь `AppError('VALIDATION_ERROR', …, 400)`.
2. ~~**`logLlmCall` без await/sideEffect**~~ — `backend/src/api/v2/modules/nightcall/llm.js` обёрнут в `sideEffect('ncl_log_llm_call', …)`.

## ~~Nightcall waivers — expiry~~ — ЗАКРЫТО (2026-08-05, из code review)

Standing-waivers (`run_id IS NULL`) больше не вечные: `_v2_ncl_decisions` получил колонку `expires_at` (retrofit ALTER), `makeReleaseDecision` фильтрует waiver-запрос `AND (expires_at IS NULL OR expires_at > NOW())`, `ncl_waive` принимает опциональный параметр `expires_at`.
- Файл: `backend/src/api/v2/modules/nightcall/{schema.js,evidence.js}`

## Nightcall proposals — терминальный apply_failed (2026-08-05, финальное ревью)

`applyExtractionIr` нетранзакционен: сбой посреди применения оставляет частично созданные accepted-требования и proposal в терминальном `apply_failed` (ретрай невозможен, optimistic-lock предотвращает двойное применение — trade-off задокументирован в коде). Восстановление — ручное. Решать вместе с транзакционной моделью модуля.
- Файл: `backend/src/api/v2/modules/nightcall/proposals.js`
- Приоритет: низкий

---

## TD-024: MCP confirm_action(approved=false) выполняет действие вместо отмены

**Обнаружено:** 2026-08-05, воркспейс trytofly.

**Проблема:**
`create_automation` вернул "REQUIRES CONFIRMATION". Последующий вызов `confirm_action(approved=false)` (явный ОТКАЗ) — автоматизация всё равно **создалась** (id=3, ответ: «Автоматизация создана»). Отклонение pending-действия исполняет его вместо отмены. Критично для HITL-флоу: подтверждение деструктивных операций (delete, bulk_delete, schema change) не защищает — отказ равен согласию.

**Где искать:**
- MCP confirm flow: `mcp-server/index.js` (обработка confirm_action) и/или backend pending-actions store
- Проверить: передаётся ли `approved` до executor'а, или pending action исполняется на любой вызов confirm_action

**Проверка после фикса:**
1. Вызвать TIER_HIGH тул → получить REQUIRES CONFIRMATION
2. `confirm_action(approved=false)` → действие НЕ выполнено, pending очищен
3. `confirm_action(approved=true)` на новом действии → выполнено

**Приоритет:** высокий — дыра в safety-механизме всех деструктивных MCP-операций.

---

## TD: upsertConfig затирает custom_domain при апдейте без него

**Дата:** 2026-08-05 · **Источник:** ревизия при фиксе NOT NULL(active) в `portal/service.js:upsertConfig`

`ON CONFLICT DO UPDATE SET custom_domain = EXCLUDED.custom_domain` — любой вызов
`upsertConfig`/`set_portal_config` без `custom_domain` пишет NULL и сбрасывает
кастомный домен воркспейса. Не чинилось вместе с active, потому что «explicit
null = очистить домен» — возможно, ожидаемая семантика какого-то вызова;
нужно решить контракт: undefined → не трогать (COALESCE), явный '' → очистить.

**Приоритет:** средний — стреляет только у воркспейсов с кастомным доменом.

---

## TD-025: Портальные пути чтения/записи в обход row-level rules

**Дата:** 2026-08-06 · **Источник:** аудит прав воркспейса fund-demo

Правила `_v2_row_rules` применялись только в основном API. Портал их игнорировал полностью: пользователь с READ-грантом на таблицу видел все строки, включая чужие. Починено **только** для `GET /:db/portal/api/tables/:typeId/objects` и для `portalUpdateObject`/`portalDeleteObject` (мост `portal/row-rules.js`, коммиты `dd81fd0a6`, `4413af327`).

Остались в обход правил:
- портальные отчёты (`portal/router.js`, ~строка 813) — синтетический user собирается с `username: email || 'portal_public'` вместо `resolvePortalUsername`;
- каталог — `portal/api.js`: `getCatalog` / `getCatalogItem` читают EAV напрямую;
- `portal/private-api.js` — заказы, документы, профиль, метрики: фильтруют по `clientObjectId`, но правила не применяют;
- агентские и MCP-прокси — `meta-kb-proxy.js`, `decisions-proxy.js`, `agent-registry-proxy.js`, `portal-agent-ctx.js`;
- `GET /api/tables/:typeId/schema` — проверяет только грант типа, `DENY_ALL`/`ROLE_ONLY` на READ не скрывают структуру таблицы.

**Как чинить:** переиспользовать `buildPortalRowFilter` / `checkPortalRowPermission` из `portal/row-rules.js`. Идентификатор автора обязан браться из `resolvePortalUsername` — иначе OWNER_ONLY разъедется между путями чтения и записи. Ветка «нет идентификатора» должна быть fail-closed: подстановка пустой строки совпадёт с `created_by = ''` и превратится в утечку.

**Приоритет:** высокий — это раскрытие чужих данных, а не косметика.

---

## TD-026: `FILTER`-правило не умеет сравнивать с атрибутом текущего пользователя

**Дата:** 2026-08-06 · **Источник:** настройка ролей fund-demo

`utils/row-permissions.js` поддерживает четыре правила: `OWNER_ONLY` (сравнение `_v2_autofields.created_by` с username), `ROLE_ONLY`, `FILTER` (реквизит против **константы** из `config {reqId, value}`) и `DENY_ALL`. Сценарий «пользователь видит записи, где реквизит Email равен его собственному email» невыразим: подстановки текущего пользователя в `FILTER` нет.

Обходной путь сейчас — `OWNER_ONLY`, но он работает только для записей, созданных этим же пользователем, и бесполезен для импортированных или заведённых сотрудником данных.

**Как чинить:** плейсхолдеры в `config.value` (`{{user.email}}`, `{{user.id}}`, `{{user.username}}`), подставляемые в `buildRowRuleWhere` параметром, а не конкатенацией.

**Приоритет:** средний — обходится, но обход хрупкий.

---

## TD-027: Заседание ИК (meta-kb debate) идёт ~17 минут

**Дата:** 2026-08-06 · **Источник:** сквозной прогон fund-demo, проект 408, 7 агентов

Тайминг живого заседания: позиции экспертов ~2 мин, кросс-дебат ~13 мин, синтез модератора и финальный ответ оркестратора — остальное. Для демонстрации и для любого интерактивного сценария это слишком долго; пользователь всё время смотрит на стрим.

Что стоит рассмотреть:
- кросс-дебат сейчас последовательный по агентам — распараллелить там, где реплики не зависят друг от друга;
- `MAX_DEBATE_ROUNDS = 5` на агента (`meta-kb/service.js:20`) — уточнить, нужен ли такой запас, когда у агента 8 инструментов;
- оркестратор после `mk_start_debate` ещё раз перечитывает огромный результат тула, чтобы «кратко пересказать консенсус» — этот проход можно убрать, консенсус уже посчитан;
- режим «быстрое заседание» с сокращённым составом.

**Приоритет:** средний — не баг, но упирается в применимость фичи.

---

## TD-028: Разный формат даты в EAV — ISO против epoch

**Дата:** 2026-08-06 · **Источник:** лента событий fund-demo

Одно и то же поле даты пишется в двух форматах: автоматизации через `run_script` кладут ISO-строку (`2026-08-04T07:30:00`), портальные server functions — epoch-секунды (`1785963557`). Читатели вынуждены разбирать оба варианта (`fmtDate` в портальном коде, дедупликация в скрипте мониторинга). Любой новый потребитель, который об этом не знает, молча сломается — например, сортировка по строке смешает форматы.

**Как чинить:** зафиксировать один формат хранения дат в EAV и привести к нему запись во всех путях (`objects`, портальный write-api, `run_script`-хелперы), плюс миграция существующих значений.

**Приоритет:** средний — тихий источник расхождений.

---

## TD-029: портальный API таблиц резолвит любое число как ссылку

**Дата:** 2026-08-06 · **Источник:** приёмка демо-портала fund-demo

`GET /:db/portal/api/tables/:typeId/objects` подставляет вместо числового значения имя объекта, если в воркспейсе найдётся объект с таким же id. Воспроизведение на `fund-demo`, таблица «Транши» (312), колонка «Номер»:

| Запись | В базе | Отдаёт API |
|---|---|---|
| Транш 1 — АвиаЛогик (467) | `1` | `"Объект (id:1)"` |
| Транш 2 — АвиаЛогик (473) | `2` | `"2"` |

Единица совпала с id корневого объекта воркспейса — значение подменилось. У двойки совпадения не случилось, и она вернулась как есть. То же самое ловится на TRL (`4` → `"Дата и время (id:4)"`) и суверенности (`8` → `"Строка (id:8)"`).

Это значит, что **значение числового поля зависит от наличия постороннего объекта с тем же id**: сегодня «4» вернётся числом, а после создания записи с id 4 — чужим именем. Клиенты вынуждены разбирать строку обратно (в fund-app для этого живёт `numVal()`), и любой новый потребитель API об этом не знает.

**Как чинить:** резолвить ссылку только для колонок, тип которых действительно ссылочный (тип реквизита известен на сервере — он уже загружается для маппинга алиасов), а не по факту «значение похоже на число». Заодно проверить тот же путь в отчётах и каталоге.

**Приоритет:** высокий — тихая подмена данных в публичном API, воспроизводится на любом воркспейсе.
### Дополнено 07.08.2026: путей сравнения даты ЧЕТЫРЕ, закрыто два

Оценка радиуса при первой записи была занижена. Мест, где даты сравниваются, четыре:

| # | место | состояние |
|---|---|---|
| 1 | `objects/service.js` → `dateSortExpr` | **закрыто** |
| 2 | `ai/agent/tools/objects.js` → сортировка в JS | **закрыто** |
| 3 | `utils/report-engine.js` | **открыто** — сортирует по `"alias".val` как по тексту, про даты не знает вовсе |
| 4 | `utils/filter-dsl.js` → `normalizeDateValue` | **открыто** — приводит к epoch только значение фильтра, а не хранимое |

**Следствие пункта 3 действует сегодня: любой отчёт, отсортированный по колонке даты со смешанными формами, сортируется неверно.** Это третий генератор SQL с другой моделью колонок; правка требует отдельной оценки.

Закрытые два идут через общий `backend/src/api/v2/utils/date-sort.js` (`dateSortKey` для JS, `dateSortSql` для SQL), ключи сверены между собой на 22 значениях против настоящего PostgreSQL.

**Расхождение подтверждено ещё раз, на других данных:** в `fund-demo` колонка «Дата» таблицы решений держит epoch у записей, созданных серверной функцией, и ISO-строку у посевных. Серверная функция отдаёт корректную `YYYY-MM-DD` — **epoch делает платформа при записи.**

**Два побочных дефекта того же класса, найденных при разборе:**
- Старый `dateSortExpr` **ронял список пятисоткой**: ветка `ELSE CAST(NULLIF(val,'') AS double precision)` на значении `01.02.2026` или `н/д` давала «неверный синтаксис для типа double precision». Закрыто.
- Ветка `YYYYMMDD` в `toTimestamp` была **недостижима** — стояла после проверки `num > 100000`, которая берёт любое восьмизначное число, и `20200408` читалось как август 1970 года. Поднята выше; **это изменение поведения записи, откатывается одним коммитом.**
- `filter-dsl.js` содержит **копию** `toTimestamp` с теми же двумя дефектами. Не тронуто.

---

## TD-029: На боевом сервере висит чужое рабочее дерево с занятой веткой

`/opt/integram/git-repos/uav/1.git` держит зарегистрированное рабочее дерево `/tmp/uav-edit` на ветке `main`, **не отцепленное**. Создано 17.07.2026, там же последний коммит.

Внутри незакоммиченная правка `MetaKbView.vue` (+17/−48) — переключатели инструментов агента в панели настроек. Заплатка сохранена: `integram-new:/root/uav-edit-2026-08-07.patch`.

**Чем плохо:** коммит в этом дереве двигает `refs/heads/main` напрямую, мимо сравнения ссылки и хука `pre-receive`. Модуль codespace везде создаёт рабочие деревья с `--detach` (правило записано в шапке `codespace/service.js`), так что слияния не блокируются — но в данных правило нарушено.

**Опасность низкая:** чтобы этим воспользоваться, нужен доступ к оболочке на боевом сервере, а у кого он есть, тот и так может всё.

**Что сделать:** выяснить у автора судьбу правки, потом `git -C /opt/integram/git-repos/uav/1.git worktree remove --force /tmp/uav-edit`. Заодно проверить, не появились ли такие ещё:

```bash
for d in /opt/integram/git-repos/*/*.git; do
  n=$(git -C "$d" worktree list 2>/dev/null | tail -n +2 | wc -l)
  [ "$n" -gt 0 ] && echo "$d: $n" && git -C "$d" worktree list | tail -n +2
done
```

**Приоритет:** низкий — гигиена, не баг.

---

## TD-030: Номера конкретного воркспейса зашиты в коде платформы

**Дата:** 2026-08-07 · **Источник:** разбор заклинившего охранника отгрузок

Правило проекта — данные воркспейса (номера таблиц, реквизитов, статусов) в коде платформы не хранятся, а разрешаются во время работы из EAV или настройки. Нарушено в девяти местах; одно закрыто, восемь открыты.

| файл | что зашито | состояние |
|---|---|---|
| `automations/execution.js` (охранник отгрузок) | `ctx.changedReqIds.includes(378)` | **закрыто** — читается `orderStatusReqId` из настройки коннектора |
| `portal/router.js:2100` | `changedReqIds: [363]` | открыто |
| `portal/router.js:2288` | `changedReqIds: newStatus ? [363, 378] : [363]` | открыто |
| `portal/telegram-callback.js:281` | `reqId === 378 && (numValue >= 395 \|\| numValue === 399)` | открыто |
| `portal/telegram-callback.js:1269` | то же выражение второй раз | открыто |
| `portal/telegram-callback.js:1616-1618` | `reqMap[338]`, `reqMap[346]`, `reqMap[423]` — название, вес, телефон заказа | открыто |
| `portal/telegram-callback.js:3387` | `objTypeId === 310` | открыто |

**Чем плохо.** В воркспейсе с другими номерами такой код молча не делает ничего: у закрытого случая охранник не срабатывал никогда, и действие не выполнялось без единого сообщения об ошибке. Это не падение, а тишина, и находится она только замером.

**Как чинить — три разных способа, путать нельзя:**
1. **Настройка уже есть** — читать из неё. Так закрыт случай с охранником: `orderStatusReqId` лежал в настройке коннектора и просто не читался.
2. **Настройки нет, но способность общая** — заводить настройку на платформе. Похоже, случай `reqMap[338/346/423]`: «название, вес, телефон» — признаки не заказа вообще, а конкретного заказа, и им место в отображении полей коннектора.
3. **Способность нужна одному воркспейсу** — выносить в код воркспейса: серверную функцию или инструмент воркспейса. С появлением действия `run_server_function` (ADR-023) это стало применимо и к автоматизациям.

**Чего делать нельзя:** заменять один номер другим. Это переносит мину, а не снимает.

**Осторожно с порядком.** Семь из восьми открытых мест — телеграм-бот на боевом сервере. Снятие хардкода меняет поведение бота у настоящих заказчиков. Сначала установить, что каждое место делает и на что опирается, и только потом трогать.

**Приоритет:** средний. Тишина вместо работы в любом чужом воркспейсе.

---

## TD-031: `execSql` калечит `?` внутри строковых литералов и jsonb-операторы

**Дата:** 2026-08-07 · **Источник:** правка сортировки дат

`mysqlToPostgres` — это `sql.replace(/\?/g, ...)` без всякого понятия о строковых литералах. Любой знак вопроса внутри кавычек считается местом подстановки параметра.

**Воспроизведено:** регулярное выражение `'^-?[0-9]{9,12}$'` ушло в базу как `'^-$2[0-9]{9,12}$'` **со сдвигом всех последующих параметров**. Обойдено на месте (`-{0,1}` вместо `-?`), но сама мина цела.

**Что ещё под ударом:** jsonb-операторы `?`, `?|`, `?&` — в PostgreSQL это операторы «ключ существует», и они будут молча искалечены.

**Чем плохо:** сдвиг параметров не даёт ни ошибки, ни предупреждения. Запрос выполняется и возвращает не то.

**Как чинить:** разбирать строку с учётом кавычек и экранирования, либо перейти на нумерованные параметры `$1..$n` в самих запросах.

**Приоритет:** высокий — молчаливое искажение данных запроса.

---

## TD-032: Числовая сортировка падает на нечисловом тексте

**Дата:** 2026-08-07 · **Источник:** тот же разбор

`objects/service.js`, сортировка по числовой колонке: `CAST(NULLIF(sort_req.val,'') AS double precision)` даёт «неверный синтаксис для типа double precision» на любом нечисловом значении в колонке — а в EAV значение хранится строкой, и попасть туда может что угодно.

Тот же дефект в сортировке по дате уже закрыт (см. TD-028): там `ELSE` заменён на `NULL` с `NULLS LAST`. Числовая ветка **не тронута** — была вне поставленной задачи.

**Как чинить:** тем же приёмом — не приводить непреобразуемое, а отдавать `NULL` и класть в конец.

**Приоритет:** средний — список падает пятисоткой, но воспроизводится только на «грязной» числовой колонке.

---

## TD-033: Индексы EAV не создаются у воркспейсов, созданных штатным путём

**Дата:** 2026-08-07 · **Источник:** трассировка вызовов + замеры плана

Воркспейс получает индексы EAV **тогда и только тогда, когда заведён скриптом миграции.** `idx_up`/`idx_t`/`idx_up_t` ставит `scripts/import-legacy-workspace.js`; `idx_up_t_ord`/`idx_t_val`/`idx_val_trgm` — `ensureEavIndexes` через `ensureAllSideTables`, единственный вызов которой в `scripts/migrate-prod-to-local.js`. **Штатно созданный воркспейс не проходит ни по одному из них.**

**Цена, замерено на копии рабочей таблицы в 94 144 строки** (дословный SQL из `objects/service.js`, медиана трёх прогонов):

| запрос | только первичный ключ | с индексами |
|---|---|---|
| страница таблицы | 38,8 мс *(Seq Scan)* | **0,44 мс** |
| счётчик | 37,7 мс | 4,1 мс |
| реквизиты страницы | 11,3 мс | 0,42 мс |
| сортировка по реквизиту | 48,0 мс | 6,0 мс |
| поиск ILIKE | **538 мс** | 6,7 мс |

Наблюдаемое следствие на деве: у единственной по-настоящему нагруженной таблицы **9,2 млн последовательных чтений и 543 млрд прочитанных строк**.

**Цена записи** (EAV пишется построчно на каждый реквизит): 20 000 строк — 75 мс без индексов, 201 мс с двумя btree, 465 мс с триграммным. Запись объекта из десяти полей дорожает на ~0,2 мс против 38 мс, экономимых на каждом открытии страницы.

**Изменения в наборе, сделанные по замерам:**
- **добавлен `idx_t_ord_id (t, ord, id)`** — порядок колонок совпадает с `ORDER BY` листинга, `LIMIT 50` останавливается на 50-й строке вместо чтения всего типа с сортировкой;
- **убран `idx_t_val (t, substr(val,1,50))`** — ни один запрос в коде не содержит этого выражения, планировщик функциональный индекс к такому предикату не применяет;
- **`idx_t_val_eq` невозможен в принципе:** запись btree ограничена ~2704 байтами, а в `val` лежат memo и документы (максимум на деве 244 226 байт). На заполненной таблице `CREATE INDEX` падает, на пустой строится и потом блокирует первую длинную запись;
- **`CREATE STATISTICS` объявлялся без схемы** → объект уходил в `public`, и `IF NOT EXISTS` делал молчаливым no-op все воркспейсы после первого: в `pg_statistic_ext` была ровно одна строка на сотни схем. После квалификации оценка выросла с 574 до 4905 при факте 8133;
- проверено и отвергнуто: **BRIN** `(up,t)` — 25,1 мс против 0,26 у btree; **`INCLUDE (val)`** — не строится, «index row requires 51368 bytes, maximum 8191».

**Выкладка на существующие воркспейсы требует `CREATE INDEX CONCURRENTLY`:** обычный держит SHARE и блокирует запись (на 382 тыс. строк — 0,27 + 0,62 + 3,2 с, дальше линейно). Написан `backend/scripts/backfill-eav-indexes.js` — идемпотентный, с `--dry-run`, `--min-rows`, `--db=`, чистит невалидные индексы от прерванных прогонов, `reltuples = -1` не путает с пустой таблицей. **Ничего не удаляет.**

**Сопровождение:** индексы распухают, нужен работающий автовакуум и при необходимости `REINDEX CONCURRENTLY`.

**Открытый вопрос:** сносить ли лишние `idx_up` / `idx_up_t` / `idx_t_val` на четырёх воркспейсах, где они уже созданы. Префиксная избыточность подтверждена планом (`idx_up_t_ord` в одиночку — 0,170/0,191 мс, с двумя добавленными — 0,194/0,192 мс), но это решение о существующих данных.

**Приоритет:** высокий — производительность каждого штатно созданного воркспейса.

---

## ~~TD-034: Стили оболочки портала красят пользовательский код~~ ЗАКРЫТ ЧАСТИЧНО

**Дата:** 2026-08-07 · **Источник:** аудит стилей портала, замер на живой странице

### Что оказалось на самом деле (первая формулировка была неверна в четырёх местах)

**Замерено, 26 разделов, 2115 пользовательских правил:**
- **Утечка внутрь (оболочка → пользовательский код): 7 селекторов** — `.portal-layout h1,h2,h3` (17 элементов), `h2` (11), `h1`, `.dot`, `.hero-cta`, `.btn-primary`, `.btn-primary:disabled`.
- **Утечка наружу: НОЛЬ на каждом разделе.** Утверждение «утечка двусторонняя» неверно: 381 из 410 пользовательских правил несут атрибут области видимости Vue, `<style scoped>` их уже изолирует.

**Голых селекторов элементов** в `portal/assets/portal.css` (1343 строки, 453 правила) было **13 правил, 3 разных селектора**: `h1` ×10, `h2` ×2, `body` ×1. Причина известна: коммит `49d993aae` свёл `<style scoped>` всех страниц в один глобальный лист, и область видимости потерялась.

`.dot`, `.price`, `.card-body`, `.card-actions`, `.height-*`, `.item-name` — **это классы, а не голые селекторы элементов**; в первой записи два разных дефекта были смешаны. Из них расходятся значениями только `.price`; у `.card-body`, `.card-actions`, `.height-*` обе копии **одинаковы**, а `.item-name` — третий случай: два компонента объявляют под одним именем **разные свойства**, и они сливаются друг в друга.

**Дублей классов 38, из них расходящихся 12; за вычетом шести законных переопределений в `@media` — 6 настоящих конфликтов.** Плюс не названный в постановке: **`@keyframes pulse` объявлен дважды с разным содержимым**, поздний молча заменял ранний, и указатель на странице входа анимировался по кривой галереи, потеряв масштабирование.

### `@scope` — проверен и **для оболочки не годится**

Поддержка: Chrome 118, Safari 17.4, Firefox 146; Baseline «newly available» с декабря 2025. Прелюдия специфичности не добавляет, голые селекторы ведут себя как `:where(:scope) sel` — проверено в браузере, `@keyframes`/`@media`/`@supports` вкладываются верно.

**Но: Firefox ESR 140 поддерживается до 29.09.2026, ESR 115 — до марта 2027, и `@scope` в них нет. Неизвестное at-правило заставляет разборщик выбросить ВЕСЬ блок целиком.** Обернуть `portal.css` в `@scope` значит показать оболочку портала на ESR **вовсе без стилей** — это отказ катастрофический, а не мягкий. **Не сделано, и делать нельзя.**

`@scope` применён **только на стороне пользовательского кода**, где он определяется на месте, а запасной путь — ровно сегодняшнее поведение.

### Что сделано

**Внутрь:** переразметка области видимости без единого риска по поддержке — все 12 осиротевших `h1`/`h2` привязаны к своим страницам, `.dot` → `.chat-dot`, `pulse` разделён на два имени, `bounce` → `chat-dot-bounce`, все 6 расходящихся дублей ограничены, блок каталога (дословно продублированный на двух страницах) сведён к одной копии. **Осталось: 1 голый селектор (`body { margin: 0 }` — законный сброс), 0 расходящихся дублей.**

**Наружу:** обёртка `.custom-code-root` с `display: contents` и `@scope` вокруг каждого внедряемого блока стилей в `CustomCodeRuntime.vue`, за проверкой поддержки. `@import`/`@charset` исключены.

### Проверка — подменой правил на живой странице, без выката

**4920 элементов на 12 разделах, по 29 вычисленных свойств. Изменилось 19:**
- **14 × `h2`** на «Отчётность LP» и «Пакет документов НТИ» — правило оболочки со страницы заказа **принудительно набирало пользовательские заголовки заглавными** с разрядкой. Видно на снимках: «СОСТАВ РАЗДЕЛА» → «Состав раздела».
- 1 × `.hero-cta` — оболочка навязывала кнопке приложения тень и переход.
- 4 × шум асинхронной загрузки.

**Ни одного изменения `color`, `background-color`, `border`, `opacity`** — то есть контраст не «перемерян», а побитово тот же. Абсолютные значения после: 132 текстовых элемента на тему, **0 ниже 4,5:1**, обе темы.

### Что осталось открытым

1. **`.btn-primary` / `.btn-secondary` остаются глобальными** (16 применений в 6 файлах) — это общий словарь кнопок оболочки, и одно живое столкновение замерено. Закрывать надо либо переименованием в пространство имён (механически, но трогает 6 шаблонов), либо `@scope`-«бубликом», который ломает оболочку на ESR. **Решение владельца.**
2. **29 неограниченных правил в пользовательском коде** (`.mark`, `.t-good`, `.t-warn`, `.t-bad`, `.num`, `.row`, `.chip-*`) сегодня ни во что не попадают только потому, что в погружённом режиме вне `.app` отрисовывается семь элементов оболочки. **На непогружённой странице они столкнутся сразу.**
3. **Прыгающая точка уже подавлена обходным путём** в прикладном коде (`animation: none` с длинным комментарием, один компонент вовсе переименован). Платформенная правка делает обходные пути ненужными, но видимого сегодня прыжка она не чинит — его нет.

**Приоритет остатка:** низкий. Основное закрыто.

---

## TD-035: Правило про импорты PrimeVue расходится с кодом

**Дата:** 2026-08-07 · **Источник:** правка автоматизаций

`.claude/rules/frontend.md` требует всегда явно импортировать компоненты PrimeVue. Проект при этом собран с `unplugin-vue-components` и `PrimeVueResolver` (`vite.config.mjs`), и, например, `ActionCard.vue` не имеет ни одного импорта PrimeVue.

Правило и код противоречат друг другу. Исполнители следуют то одному, то другому — код получается разнородным.

**Что сделать:** решить, какая сторона верна, и привести к ней вторую. Оба варианта рабочие, важна однозначность.

**Приоритет:** низкий — гигиена, не баг. Но каждый новый исполнитель спотыкается заново.

---

## TD-036: Прозрачность на недоступных органах управления в оболочке портала

**Дата:** 2026-08-07 · **Источник:** замер экрана мониторинга фонда

`portal/` объявляет приглушение недоступных элементов прозрачностью: `.btn-primary:disabled { opacity: .6 }`, `.btn-telegram:disabled { .6 }`, `.chat-send:disabled { .5 }`, плюс тема PrimeVue через `--p-disabled-opacity`.

**Формально это допустимо:** WCAG 1.4.3 выводит неактивные органы управления из-под требования по контрасту. Дефекта по букве стандарта нет, и спор об этом закрыт — исключение существует.

**Практически оно вредит там, где недоступная кнопка — главное действие экрана.** Замерено: главная кнопка экрана мониторинга фонда недоступна по умолчанию (ни один источник не подключён) и давала **2,29:1**. Исключение писалось про элемент, мимо которого проходят; здесь это единственное действие, и прочитать его нужно именно затем, чтобы понять, чего сейчас нельзя.

**Второй довод, не связанный со стандартом:** `opacity` снижает контраст **незаметно для `getComputedStyle().color`** — правило печатает исходный цвет, и провал не ловится ни чтением кода, ни поиском по нему, только замером отрисованной страницы. За сутки этим пробило шесть мест.

**Как чинить:** приглушать цветом из палитры вместо прозрачности. В прикладном коде фонда это уже сделано (`.btn:disabled`, `.btn.primary:disabled`, `.login-btn:disabled`, `.inp:disabled`, `.dg-btn:disabled`). В оболочке — нет.

**Приоритет:** низкий по стандарту, средний по существу. Затрагивает каждую недоступную кнопку портала.

---

## TD-037: `list_objects` — сортировка применяется ПОСЛЕ пагинации

**Дата:** 2026-08-07 · **Источник:** сквозной прогон инвест-комитета в `fund-demo`

Запрос `list_objects(743, { sort: "-id", limit: 5 })` вернул **пять самых СТАРЫХ записей**, упорядоченных по убыванию, при `hasMore: true`. Проверено дважды — с `limit: 5` и `limit: 10` — против контрольного `limit: 100`.

То есть страница набирается в порядке хранения, и только потом сортируется **внутри уже набранной страницы**. Запрошенные «пять последних» не имеют к последним никакого отношения.

**Чем плохо, и почему это дороже обычной ошибки:** ответ выглядит правдоподобно. Пять записей, отсортированы по убыванию — всё как просили. Обнаруживается только сверкой с полной выборкой. В этом самом прогоне дефект дал ложный вывод «событий `IC1_CONVENED` в базе нет», хотя они есть.

**Это тот же класс, что уже записан в gotchas про `where`:** отбор и сортировка работают внутри страницы, а не в запросе.

**Как чинить:** переносить `ORDER BY` в сам запрос, до `LIMIT`.

**Приоритет:** высокий — молчаливо неверный ответ на самый обычный запрос.

**ЗАКРЫТО 07.08.2026.** Место оказалось не то, которое называла постановка: `objects/service.js` (путь REST) сортировал в запросе и считал по фильтру и до правки — замерено. Дефект жил в `ai/agent/tools/objects.js` — отдельной реализации, которую зовут MCP и агент в приложении. `ORDER BY` перенесён в запрос (`buildListOrder`); сортировка по колонке берёт значение боковым подзапросом и читает **оба** способа хранения ссылки, перевёрнутый и прямой, — читался только перевёрнутый, из-за чего в `chk_log_demo` сортировка по ссылочной колонке молча не делала ничего. Замер на живой базе: `sort:'-id', limit:5` было `[1004,1003,1002,1001,1000]`, стало `[1030,1029,1028,1027,1026]` — совпадает с контрольным `limit:100`.

---

## TD-038: `list_objects` — `total`, `hasMore` и `summary` игнорируют `where`

**Дата:** 2026-08-07 · **Источник:** тот же прогон

Запрос с `where = { Код: "IC1_CONVENED" }` вернул **4 строки**, но `total: 31`, `hasMore: true` и агрегаты, посчитанные **по всей таблице**.

**Следствие:** «нашлось 4 из 31» читается как «показана первая страница из тридцати одной подходящей», хотя подходящих ровно четыре. Счётчик отвечает не на тот вопрос, который задали.

Это ровно тот механизм, который уже давал нам ложное «записей нет» и записан в `gotchas.md`. Здесь он даёт обратное — ложное «записей много».

**Как чинить:** считать `total` и агрегаты по отфильтрованному набору, либо честно назвать их «по всей таблице» в ответе и в описании инструмента. Молчаливое расхождение хуже обоих вариантов.

**Приоритет:** высокий, вместе с TD-037 — это одна причина.

**ЗАКРЫТО 07.08.2026.** Отбор (`where`/`filters`) и поиск (`search`) перенесены в SQL (`buildListWhere` в `ai/agent/tools/objects.js`); подсчёт, выборка и агрегация получают **одно и то же** условие, разойтись им негде. Оговорка «счётчик по всей таблице» не понадобилась: счёт по фильтру дешёвый, отдельный запрос `COUNT` уже был. Замер на живой базе, 31 запись, 4 подходящих: было `строк 2, total 31, hasMore true`, стало `строк 4, total 4, hasMore false`. Неизвестное имя колонки даёт `1 = 0`, а не полную таблицу.

**Осталось рядом (не тот же дефект):** `summary` собирается только по перевёрнутому способу хранения ссылки. В воркспейсах, где ссылки лежат прямым способом (`chk_log_demo`), агрегаты пусты — и были пусты до правки.

---

## TD-039: приведение EAV-текста к числу и дате роняет запрос — 22 места

**Дата:** 2026-08-08 · **Источник:** сплошной перебор приведений `val` + исполнение на проде

Значения EAV лежат в `val TEXT`. PostgreSQL на невалидном вводе **возбуждает ошибку**, а не отдаёт NULL, поэтому голое `CAST(val AS DECIMAL)` — отказ всего запроса, а не пропуск строки. MySQL, откуда перенесены эти выражения, возвращал 0 и предупреждение.

**Падает сегодня:** пересчёт ROLLUP «Состав» в `usadba_3` (`computed-reqs.js:200,512`). Приведению подвергается колонка 381 — 5877 строк, из них 8 держат текст вместо номера товара («Мёд липовый 2025, 350 мл», «Свечи 50 минут горения»). Запрос по родителям этих строк падает: `invalid input syntax for type bigint`.

```sql
SELECT count(*) FILTER (WHERE val !~ '^[0-9]+$') FROM usadba_3.usadba_3 WHERE t = 381;
```

**Ждут своего значения (21 место):**

| файл | точки | сработает, когда |
|---|---|---|
| `utils/computed-reqs.js` | 252×2, 259, 304, 577×2, 584, 685 | в цену или количество попадёт «0,5» либо «н/д» |
| `modules/portal/private-api.js` | 418, 419, 428, 429, 437, 445, 446 | в конфиге портала зададут `valueReqId`/`dateReqId`. `val::date` падает на `1779823620` — канонической форме хранения даты, то есть сломано по построению |
| `modules/ai/agent/tools/portal.js` | 234, 430, 431 | зададут `totalReqId` / `itemQtyReqId` |
| `utils/report-engine.js` | 506 | пользователь применит фильтр `>=` по числовой колонке отчёта |
| `modules/objects/service.js` | 1456 | создадут запись в таблице с уникальной числовой колонкой, где уже лежит нечисловое |

**Как чинить — не одинаково для всех.**

Обёртка `CASE WHEN val ~ '<число>' THEN CAST(...) END` годится там, где NULL — верный ответ: сортировка и диапазонные фильтры. Так уже сделано в `objects/service.js`, `views/service.js`, `filter-dsl.js` и работает.

Для **денежных агрегатов и ссылок она вредна**: вместо громкого отказа получается молча заниженная сумма заказа и ненайденная связь. Там чинить данные — запретить запись нечислового в числовую колонку и вычистить существующее, — а приведение оставить строгим.

**Индексация решается без переделки схемы.** Индекс строится по самому выражению; замер на 382 000 строк: `Index Cond`, 0,133 мс против 5,071 мс у `Filter`. Типизированные колонки для этого не нужны.

**Приоритет:** средний. Одно место падает сегодня и чинится точечно (8 значений в колонке 381); остальное — мина, а не пожар.

**Разбор:** `docs/superpowers/plans/2026-08-08-eav-cast-guards.md` — полная опись 30 точек (22 дефекта, 8 охранённых собственным условием), с воспроизведением каждой на проде. План написан под сплошную обёртку и в части денег и ссылок **не годится** — использовать как справочник по местам, не как руководство к действию.

---

## TD-040: переезд воркспейса не переносит данные и молчит об этом

**Дата:** 2026-08-08 · **Источник:** разбор двумя сессиями независимо

`PATCH /workspaces/:slug/remote-dsn` проверяет соединение, записывает строку подключения и зовёт `bootstrapWorkspaceDb` — та создаёт схему, EAV-таблицу и служебные таблицы на новой стороне. **Копирования записей нет ни одной строкой** (`workspaces/service.js:226-234`).

Данные при этом **не уничтожаются**: `getPoolForDb` просто отдаёт другой пул, записи остаются в прежней базе, снятие `remote_dsn` возвращает доступ. То есть отключение, а не потеря, и оно обратимо.

Опасно, что отключение молчит: воркспейс открывается, таблицы на месте, записей нет, и пустая таблица неотличима от таблицы, куда никто не вносил. Усугубляется тем, что `execSql` на отсутствующем отношении возвращает маркер `DB_NOT_FOUND`, а не бросает.

**Как чинить:** план `docs/superpowers/plans/2026-08-08-workspace-move-guard.md` — отказ 409 с обоими числами вместо молчания; переезд на пустую цель по явному `confirmEmptyTarget`. Перенос данных между серверами в него не входит: существующий образец (`workspace-clone.js:420`) работает `INSERT … SELECT` в пределах одного соединения, а кроме EAV пришлось бы переносить документы, блоки, версии, папки и граф.

**Приоритет:** средний. Обратимо, но выглядит как потеря.

---

## TD-041: двенадцать кэшей помечены именем базы, которое не вечно

**Дата:** 2026-08-08 · **Источник:** перебор `.set(db` по `backend/src`

Имя базы принимают за вечный и единственный признак того, куда смотрит воркспейс. Оба допущения неверны: переезд имя не меняет (`registerRemoteDsn(db_name, dsn)`, после чего `getPoolForDb` отдаёт другой пул, возможно на другом сервере), а `deleteWorkspace` сносит схему, **не чистя ни одного кэша** — следующий воркспейс с тем же именем получит чужие ответы до перезапуска процесса.

Двенадцать объявлений в девяти файлах: `workspace-tools/service.js:16,57`, `workspace-tools/sandbox.js:24`, `timeseries/service.js:21`, `ai/starters.js:42`, `ai/agent/context.js:20`, `ai/router.js:62`, `presence/service.js:47`, `swarm-memory/behavioral-collector.js:27,29`, `calls/signalling.js:7,10`.

Тише всего у проверок DDL: кэш отвечает «таблица создана», на новой стороне её нет, а `execSql` возвращает `DB_NOT_FOUND` вместо исключения.

**Образец правильного решения уже в репозитории:** `registry/lazy-init.js:52` и `utils/create-audit-table.js:65` ключуют `WeakMap` по объекту пула. Эти два файла править не надо.

**Как чинить:** план `docs/superpowers/plans/2026-08-08-db-scoped-caches.md` — фабрика вместо `new Map()` и сброс по имени на событиях `workspace.deleted` / `workspace.updated`. Оговорка в плане: шина процесс-локальная, при нескольких форках pm2 остальные сохранят записи до собственного события.

**Приоритет:** средний.

---

## TD-042: создание автоматизации принимает `active` и выбрасывает

**Дата:** 2026-08-08 · **Источник:** живая проба на проде другой сессией

Схема маршрута объявляет `active: z.boolean().optional()` (`automations/router.js:95`), колонка объявлена `NOT NULL DEFAULT TRUE` (`service.js:138`), а `createAutomation` поле **не читает**: его нет ни в разборе тела, ни в списке колонок `INSERT` (`service.js:262,268`). Значение берётся из умолчания.

Проверено пробой: послан `active: false`, ответ вернул `active: true`.

**Следствие:** завести автоматизацию выключенной нельзя. Расписание становится живым в тот же миг, когда создано, — до того, как автор успел его проверить. Обходной путь «создать и сразу погасить» оставляет окно, в котором оно может сработать.

Всё остальное на месте: `updateAutomation` поле обрабатывает (`service.js:289`), планировщик берёт только включённые (`scheduler.js:75,124`). Образец правильной записи — в том же файле сорока строками ниже: `insertSystemAutomation` (`service.js:319-326`) колонку перечисляет и ставит `FALSE` намеренно.

**Как чинить:** план `docs/superpowers/plans/2026-08-08-automation-create-active.md`. Правка в одном месте.

**Приоритет:** высокий — единственный из четырёх, где дефект приводит к незапланированному действию, а не к неверному показу.
