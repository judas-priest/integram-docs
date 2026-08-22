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

## ~~TD-024: MCP confirm_action(approved=false) выполняет действие вместо отмены~~ ЗАКРЫТ

**Закрыт 2026-08-13.** Корневая причина — JS type coercion: `!!"false" === true`. Все три точки входа (MCP `handleConfirmAction`, `/mcp-resume`, `/agent-resume`) теперь используют `approved === true` вместо `!!approved`/`?? false`. Регрессионный тест: `hitl-reject.test.js`.

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

## TD-029b: На боевом сервере висит чужое рабочее дерево с занятой веткой

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

## TD-039: приведение EAV-текста к числу и дате роняет запрос — 18 мест

**Дата:** 2026-08-08 · **Источник:** сплошной перебор приведений `val` + исполнение на проде

Значения EAV лежат в `val TEXT`. PostgreSQL на невалидном вводе **возбуждает ошибку**, а не отдаёт NULL, поэтому голое `CAST(val AS DECIMAL)` — отказ всего запроса, а не пропуск строки. MySQL, откуда перенесены эти выражения, возвращал 0 и предупреждение.

**Падает сегодня:** пересчёт ROLLUP «Состав» в `usadba_3` (`computed-reqs.js:315`; до слияния одиночного вычислителя с пакетным — 200 и 512). Приведению подвергается колонка 381 — 5877 строк, из них 8 держат текст вместо номера товара («Мёд липовый 2025, 350 мл», «Свечи 50 минут горения»). Запрос по родителям этих строк падает: `invalid input syntax for type bigint`.

```sql
SELECT count(*) FILTER (WHERE val !~ '^[0-9]+$') FROM usadba_3.usadba_3 WHERE t = 381;
```

**Ждут своего значения (17 мест):**

| файл | точки | сработает, когда |
|---|---|---|
| `utils/computed-reqs.js` | 380×2, 387, 488 | в цену или количество попадёт «0,5» либо «н/д» |
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

---

## ~~TD-043: PM-триггеры автоматизаций не работали~~ ЗАКРЫТ

`dispatchPmTrigger` выбирал подходящие автоматизации и **только логировал** их — комментарий в самом файле признавал, что исполнение «сложное» и не сделано. Четыре PM-триггера (`on_issue_created`, `on_issue_status_changed`, `on_sprint_started`, `on_sprint_completed`) поэтому молча не срабатывали. Вдобавок этих типов не было в `TRIGGER_TYPES`, питающем `z.enum` при сохранении, — то есть такую автоматизацию нельзя было даже создать.

**ЗАКРЫТО 2026-08-07:** диспетчер делегирует в `dispatchEvent` — тот же путь, которым идут `object.*`; четыре типа добавлены в whitelist. Найдено при разборе техдолга Nightcall (TD-NCL-16 ссылался на этот файл как на образец).

---

## TD-044: портал закрывается только рисунком — данные остаются открыты анониму

**Файлы:**
- `portal/server/middleware/portal-auth.ts:11` — `PROTECTED_SUFFIXES`
- `backend/src/api/v2/modules/portal/router.js` — публичные ручки `/api/tables/...`, `/api/catalog`, `/api/reports/:id`
- `backend/src/api/v2/modules/portal/roles.js:120` — `filterByRole`

**Проблема:**
Способа закрыть портал целиком нет. SSR-сторож знает ровно четыре пути — `/orders`, `/profile`, `/documents`, `/support`; главной среди них нет. Роли страниц (`page.roles`) применяются только в `/auth/me` как подсказка клиенту (`router.js:1140`) и завязаны на запись клиента в EAV — у портала без таблицы клиентов их нет вовсе.

Поэтому закрытие пишут внутри кастомного компонента: он зовёт `/auth/me`, и до успешного ответа не рисует содержимое и не запрашивает данные. Так сделано в кабинете сотрудников усадьбы (`main/StaffApp.vue`, репо `portal-components` воркспейса `usadba_3`) и так же — в «РУБЕЖ» (`economic-simulator.vue`, репо `portal` воркспейса `posoh_monitor`, ветка `release`, коммит `689484e` от 13.08.2026).

**Чего это не закрывает:** данные. Проверено анонимным запросом 13.08.2026 — куки нет, ответ 200:

```
GET https://drondoc.online/api/v2/posoh-monitor/portal/api/tables/5753/objects?limit=2
{"ok":true,"data":{"items":[{"id":5771,"name":"Курская нефтебаза (Курск)","fields":{"Ущерб при полном поражении, руб":"1500000000", ...
```

Гейт в компоненте прячет экран, но каждая таблица, попавшая в `bindings` раздела, читается по прямому адресу кем угодно. Для «РУБЕЖ» это девять справочников, включая профили объектов с координатами и оценкой ущерба.

**Мёртвые ключи конфига.** Двум порталам флаг уже прописан, и оба не работают — ключи не читает никто (проверено грепом и в master, и в выложенном `/opt/integram/backend`):
- `posoh_monitor`: `{"requireAuth": true}`
- `plm_gift`: `{"methods":["email_otp"],"allowRegistration":false}`

Живые ключи секции `auth` — только `clientsTypeId`, `phoneReqId`, `nameReqId`, `tgChatIdReqId`, `tgUsernameReqId`, `roleReqId`, `grantsConfig`. Имя `requireAuth` в коде встречается (`portal/auth.js:728`), но это локальная переменная `portalAuth()`, с конфигом не связанная, — грепом по имени это выглядит рабочим флагом, и один раз уже ввело в заблуждение.

**Решение:** читать `config.auth.requireAuth` в двух местах — middleware в начале портального роутера (закрывает API; открытыми оставить `/api/config`, `/api/auth/*`, `/api/files/*`) и SSR-сторож портала (закрывает страницы; открытой оставить `/auth`). Нужны оба: бэкенд на проде не видит запросов страниц — nginx (`/etc/nginx/sites-enabled/drondoc.online:37`) отдаёт их Nuxt напрямую, минуя SSR-прокси. Мёртвые ключи `methods`/`allowRegistration` при этом либо реализовать, либо убрать из конфига `plm_gift`.

**Приоритет:** средний. Дефекта в работе нет — но владелец, поставивший `requireAuth: true`, считает портал закрытым, а он открыт целиком.

---

## TD-045: страница отказа в доступе показывает поверх себя технические ошибки тех же запросов

**Файлы:**
- `frontend/src/services/api.js:51-58` — перехватчик 403
- `frontend/src/main.js:75-77` — `onWorkspaceAccessDenied` → `router.push({ name: 'access-denied' })`
- `frontend/src/views/ai/AiChat.vue:243`, `:955` — тосты компонента

**Проблема:**
Перехватчик на 403 «Workspace access denied» уводит на `/access-denied` **и** отклоняет промис. Отклонение доезжает до компонента, тот показывает свой тост. Компонент, пославший несколько запросов, показывает несколько тостов, и каждый ответ 403 вдобавок повторяет `router.push`.

Наблюдалось на проде 12.08.2026: под сессией без членства в воркспейсе открыт `https://ai2o.online/soric_demo/ai?mode=metakb` — страница `/access-denied` с текстом «У вас нет прав для доступа к этому рабочему пространству», а поверх неё два тоста: «Ошибка загрузки чата: Request failed with status code 403» и «Meta KB: Request failed with status code 403».

Читается это плохо вдвойне. Во-первых, объяснение уже дано страницей, а тосты сообщают то же самое второй раз и хуже. Во-вторых, текст тоста — сырое сообщение axios: `e.message` берётся напрямую (`AiChat.vue:243,955`), хотя в `api.js:169` есть словарь `403: 'Нет доступа'`, а в `useAppToast().error()` — санитизация через `sanitizeApiError`.

**Решение:** после того как перехватчик признал 403 отказом в доступе к воркспейсу, отклонять промис особой причиной (например `cancelledError`-подобной пометкой), а в компонентах не показывать тост на такую причину — как уже сделано с отменёнными запросами (`config.signal?.aborted` в `api.js:49`). Заодно перевести оба тоста `AiChat.vue` на `useAppToast().error(e)`, чтобы наружу не выходило `Request failed with status code N`.

**Приоритет:** низкий. Доступ закрыт правильно, страница отказа показывается — портится только то, как отказ выглядит.

---

## TD-046: агрегаты вычисляемых реквизитов складываются в JS, а не в базе

**Дата:** 2026-08-13 · **Источник:** обзор PR #139 (слияние двух вычислителей) + замер на проде

`evalComputedReqsBatch` для SUM/AVG/MIN/MAX выбирает каждую строку-ребёнка и складывает её в JS:

```js
// utils/computed-reqs.js:387 (и :488 для ветки linkReqId)
`SELECT up, CAST(val AS DECIMAL(20,6)) AS v FROM ${safeDb} WHERE up IN (${ph2}) AND t = ?`
...
if (fn === 'SUM') result[oid][`computed_${def.id}`] = vals.reduce((a, b) => a + b, 0);
else if (fn === 'AVG') result[oid][`computed_${def.id}`] = vals.reduce((a, b) => a + b, 0) / vals.length;
```

Двоичный float для денег — то, чего не делают ни в легаси, ни в справочниках по хранению валют: в `crm/index.php:3092,3396` вычисляемые поля считаются целиком в SQL (`sum|avg|count|min|max|group_concat` внутри SELECT, вокруг собирается GROUP BY). До слияния вычислителей одиночный путь тоже агрегировал в SQL — при унификации осталась JS-ветка, то есть худшая из двух.

**Сегодня не проявляется.** Прогон по всем позициям `usadba_3` (SUM цены×количества, реквизиты 362 и 361): 2861 заказ с позициями, **0 расхождений** между десятичной суммой SQL и суммой во float. Данные целые, хвостов нет. AVG-роллапов на проде нет вовсе.

**Когда выстрелит:** дробное количество (0,5 кг), процентная или курсовая колонка в `targetColId`, длинный список позиций. `0.1 + 0.2` даёт `0.30000000000000004`; AVG вместо `2.333333` покажет `2.3333333333333335`.

**Решение:** заменить построчную выборку на агрегацию в запросе — `SELECT up, SUM(CAST(val AS DECIMAL(20,6))) ... GROUP BY up`. Запрос остаётся один, кода становится меньше.

**Что при этом сломается и почему это не сделано сразу:** `numeric` приходит из pg строкой (`"65800.000000"`), а сейчас в ответе число. Вид значения изменится разом на всех поверхностях — таблица, отчёты, выгрузки, портал, — и потребует правки форматирования на клиенте (`CellRenderer`, тип 15 `CALCULATABLE` печатает значение как есть). Делать вместе с выводом типа виртуальной колонки из определения (SUM по денежной колонке → CURRENCY), иначе форматирование останется прибитым к одному типу.

**Приоритет:** низкий, пока в агрегируемых колонках нет дробных значений. Проверка, что момент настал:

```sql
SELECT count(*) FROM <ws>.<ws> WHERE t = <targetColId> AND val ~ '\.[0-9]';
```

---

## ~~TD-047: объявленный тир инструмента воркспейса не участвует ни в правах, ни в показе~~ ЗАКРЫТ

**Закрыт 14.08.2026.** Тир из строки `_v2_workspace_tools` доезжает до правил доступа двумя путями (`listWorkspaceToolDefs` и сборка агента воркспейса) и применяется одинаково на всех ТРЁХ заслонах вызова: `execOneTool` в исполнителе, обёртка воркспейсных у оркестратора и общий `portalGrantsMiddleware` в `PRE_CHECK_ORDER` — последний берёт тир из карты `ctx._wsTierByName`, потому что перечня записей внутри `executeTool` уже нет. Платформенную классификацию довод не перекрывает: он действует только для имён, у которых нет СВОЕЙ строки в `TOOL_TIERS`, а полнота этого перечня закреплена сторожем `tool-tiers-coverage.test.js` (582 имени, без строки — ноль). Заодно закрыты две дыры, из-за которых доверять полю было нельзя: `updateTool` не пересчитывал тир по способностям (в отличие от заведения и импорта), а `toolRiskTier` брал значение у прототипа — `toolRiskTier('constructor')` возвращал функцию, которая роняла вставку записи в журнал вызовов целиком.

**Цена, названная вслух:** инструмент воркспейса, объявленный первым тиром, теперь доступен портальному вызывающему без грантов. Это замысел — тир объявляет владелец воркспейса, — но это расширение доступа, а не только починка.

**Проверено на живых данных 14.08.2026:** в `uav` на integram-new все активные строки объявлены первым тиром при пустых способностях, то есть пересчёт Задачи 1 ничего не переклассифицирует задним числом.

Ниже — описание долга, каким оно было до закрытия.

Строка `_v2_workspace_tools` несёт `risk_tier` (умолчание 1, проверяется при записи, доезжает до `getCachedDefs`), но **на пути исполнения этим полем не пользуется никто**. Сплошной поиск по `backend/src` 13.08.2026: единственное употребление вне записи и кеша — проверка в тесте службы.

Права портального вызывающего решает `checkPortalGrantForTool`, а тир он берёт у платформенного классификатора:

```js
export function toolRiskTier(toolName) {
  return TOOL_TIERS[toolName] ?? TIER_MEDIUM; // unknown tools default to MEDIUM
}
```

Инструментов воркспейса в `TOOL_TIERS` нет по построению — они заводятся данными, а не кодом. Значит **любой** из них считается пишущим, даже объявленный читающим калькулятором. Портал без выданных грантов не сможет ни увидеть такой инструмент (после отбора по правам, 13.08.2026), ни вызвать его (запрет на вызове был и раньше).

Проверить, что тир расходится:

```bash
cd backend && node --input-type=module -e "
import { toolRiskTier } from './src/api/v2/modules/ai/agent/risk-tiers.js';
console.log(toolRiskTier('swarm_economics'));  // 2 — платформа не знает имени
"
# при этом в базе воркспейса у него объявлен первый тир:
# select name, risk_tier from uav._v2_workspace_tools where name = 'swarm_economics';  → 1
```

**Сегодня это никого не задевает, и это проверено, а не предположено.** На стенде nti-new четыре портала с включённым агентом: у `uav` гранты полные (`allowWrite/allowDelete/allowSchema` все истинны, плюс то же в `roleOverrides.staff`) — там ничего не отсекается; у `udmfic`, `posoh_monitor`, `sppr_demo` воркспейсных инструментов нет вовсе (0, 0, 0).

**Проявится** на первом же портале, где гранты не выданы, а воркспейсные инструменты заведены: калькуляторы станут невидимы агенту без единого признака на экране.

**Лечение:** дать `checkPortalGrantForTool` необязательный тир и передавать его там, где имя приходит из строки воркспейса — `listWorkspaceToolDefs` (сейчас отдаёт `name`, `group`, `description`, `parameters`, `_workspace`) и сборка агента воркспейса. Трогать при этом придётся путь прав, поэтому правку делать отдельно и с проверками на обе стороны: и показ, и вызов.

**Приоритет:** низкий до появления такого портала; ловится запросом

```sql
SELECT db FROM _v2_portal_config
 WHERE config->'agent'->>'enabled' = 'true' AND config->'agent'->'tools' IS NULL;
```

---

## ~~TD-048: смена родителя задачи проверяется без блокировки строк~~ ЗАКРЫТ

**ЗАКРЫТО 14.08.2026.** Смена родителя идёт под консультативной блокировкой на рабочую область — `SELECT pg_advisory_xact_lock(hashtext('pm_reparent:<db>'))` внутри транзакции. Построчная (`FOR UPDATE` по цепочке предков), как предлагала запись, не годится: у двух одновременных перевесов «A под B» и «B под A» цепочки идут в РАЗНЫЕ стороны, и строки захватывались бы в обратном порядке — вместо цикла в данных вышла бы взаимная блокировка. `updateIssue` открывает транзакцию только когда правка касается родителя (обычная правка идёт как раньше), `bulkUpdate` берёт ту же блокировку — повторный захват внутри одной транзакции проходит, поэтому массовый путь себя не блокирует. Оба меньших пропуска закрыты там же: обход, вышедший по счётчику 32 шагов, теперь отвечает 400 «Parent hierarchy is too deep or cyclic», а не разрешает перевес; `subtreeHeight` считает и удалённых потомков (`UNION` вместо `UNION ALL` — на данных с циклом рекурсия иначе не сходится), поэтому восстановление ветки из корзины больше не может дать глубину выше `MAX_DEPTH`. План — `docs/superpowers/plans/2026-08-14-pm-td-048-051.md`, задачи 4–6; проверки — `pm/__tests__/reparent-lock.test.js`.

**Дата:** 2026-08-14 · **Источник:** разбор задачи 5 плана `docs/superpowers/plans/2026-08-13-pm-orgs-tails.md`

`assertReparentAllowed` (`backend/src/api/v2/modules/pm/service.js:96`) читает цепочку предков, решает и пишет — всё вне транзакции и без `FOR UPDATE`. Два одновременных запроса «A под B» и «B под A» оба проходят проверку по старому состоянию и вместе создают цикл — ровно тот, который проверка и запрещает. Массовый путь идёт в транзакции, но `SELECT` там без блокировки, так что защищает только сериализация внутри одного запроса.

Рядом два меньших пропуска той же проверки:
- обход вверх ограничен 32 шагами; на данных с уже существующим циклом, не содержащим перевешиваемую задачу, обход выходит по счётчику молча — перевес разрешается;
- `subtreeHeight` и `getDepth` фильтруют `deleted_at IS NULL`, поэтому поддерево под мягко удалённым потомком в расчёт глубины не попадает: перевес проходит, а восстановление вернёт ветку и даст глубину больше `MAX_DEPTH` мимо всякой проверки.

**Решение:** блокировать участвующие строки (`SELECT ... FOR UPDATE` по цепочке предков) внутри транзакции; при выходе обхода по счётчику отвечать ошибкой, а не разрешением.

**Приоритет:** низкий. Требует одновременных правок иерархии одной и той же ветки — на нынешней нагрузке не наблюдалось.

---

## ~~TD-049: пункт чек-листа адресуется номером позиции, а не своим признаком~~ ЗАКРЫТ

**ЗАКРЫТО 14.08.2026.** У пункта появился устойчивый признак — поле `id` (UUID) внутри элемента JSONB. Признак выдаётся при создании: `addChecklistItem`, `createIssue` и клонирование повторяющейся задачи (там — НОВЫЕ признаки, скопированный жил бы сразу в двух задачах). Старым спискам признаки проставлены разово в `ensureTables`; наличие поля проверяется через `->>'id' IS NULL`, а не оператором `jsonb ? 'id'` — `execSql` считает знак вопроса местом подстановки. По признаку адресуются оба пути: REST (`PATCH /pm/issues/:id/checklist` с телом `{ itemId, done }`, `DELETE /pm/issues/:id/checklist/:itemId`) и инструмент ИИ (`pm_toggle_checklist` принимает `item_id`); интерфейс `IssueChecklist.vue` — тоже. Второй, неатомарный путь закрыт: запись `checklist` целиком через `updateIssue` отвергается с 400 (значит, и `PATCH /pm/issues/:id`, и `bulkUpdate`), а `pm_update_issue` поле не пробрасывает вовсе. План — тот же, задачи 7–11; проверки — `pm/__tests__/checklist-item-id.test.js`, `frontend/src/components/pm/__tests__/IssueChecklist.spec.js`.

**Дата:** 2026-08-14 · **Источник:** разбор задач 7 и 19 того же плана

Отметка, добавление и удаление пункта теперь атомарны в базе (`toggleChecklistItem`, `addChecklistItem`, `removeChecklistItem`), но пункт адресуется НОМЕРОМ ПОЗИЦИИ. Если сосед удалил пункт выше по списку, галочка ляжет на съехавший пункт, и ответ будет 200 — потери правки нет, но отмечено не то.

Второй путь остался неатомарным целиком: `updateIssue` принимает поле `checklist` и пишет массив целиком (`service.js:374`), через него ходят `PATCH /pm/issues/:id`, `bulkUpdate` и инструмент ИИ `pm_update_issue`.

**Решение:** дать пункту устойчивый признак (`id` внутри элемента JSONB) и адресовать правки по нему; запретить запись `checklist` целиком через `updateIssue`, оставив только операции уровня пункта.

**Приоритет:** средний, если чек-листами пользуются вдвоём одновременно; низкий иначе.

---

## ~~TD-050: живые подзадачи под мягко удалёнными родителями из старых данных~~ ЗАКРЫТ

**ЗАКРЫТО 14.08.2026.** Из двух вариантов выбран первый: живые подзадачи под удалённым родителем перевешиваются в корень — запись остаётся видимой и перестаёт ссылаться на удалённое. Правка идёт разово при создании таблиц (`ensureTables`, метка `pm_fix_orphans`), рядом с прочими идемпотентными миграциями: они уже выполняются лениво, один раз на рабочую область, повторный прогон не находит ничего. Оба следствия каскада закрыты там же: `restoreIssue` шлёт `pm.issue.updated` на КАЖДУЮ поднятую задачу с автором правки (автор передаётся и маршрутом, и инструментом ИИ `pm_restore_issue`), а сброс кэша организации свёрнут по рабочей области окном 200 мс — дерево из N задач даёт один обход и один сброс вместо N. План — тот же, задачи 1–3; проверки — `pm/__tests__/orphan-fix.test.js`, `pm/__tests__/restore-events.test.js`, `orgs/__tests__/cache-debounce.test.js`.

**Дата:** 2026-08-14 · **Источник:** ревью задачи 6 того же плана

Каскад удаления обрывается на уже удалённом узле (`service.js:459`, `WHERE c.deleted_at IS NULL`). Данные, созданные ДО каскада, могут содержать: родитель удалён поодиночке, живой ребёнок ссылается на него. Удаление прародителя такую ветку не подберёт, а `restoreIssue` теперь откажется поднимать подзадачу, пока родитель в корзине, — то есть запись оказывается запертой.

**Решение:** разовая миграция — найти `parent_id`, указывающие на строки с `deleted_at IS NOT NULL`, и либо перевесить их в корень, либо отправить в корзину вслед за родителем.

```sql
SELECT c.id, c.number, c.parent_id FROM <ws>._v2_pm_issues c
  JOIN <ws>._v2_pm_issues p ON p.id = c.parent_id
 WHERE c.deleted_at IS NULL AND p.deleted_at IS NOT NULL;
```

Заодно два мелких следствия каскада: `restoreIssue` не шлёт событий вовсе, поэтому возвращённое поддерево не появится на чужих досках до перезагрузки; сброс кэша организации (`orgs/pm-listeners.js:11`) подписан на каждое событие, и дерево из N задач даёт N сбросов вместо одного.

**Приоритет:** низкий, объём определяется запросом выше.

---

## ~~TD-051: экран метрик PM молчит на пустых данных~~ ЗАКРЫТ

**ЗАКРЫТО 14.08.2026.** Три блока, которые молчали, теперь объясняют пустоту вместо пустых осей и голой шапки: без завершённых спринтов вместо столбчатого графика — «Нет завершённых спринтов — скорость считать не по чему»; без задач вместо накопительной диаграммы — «Задач пока нет — накопительной диаграмме нечего показывать»; без назначенных задач таблица загрузки показывает через `#empty` «Никому ничего не назначено». Признаки пустоты (`hasVelocity`, `hasCfd`) считаются по наличию наборов, а не по нулям, — «нет данных» и «данные нулевые» на экране больше не совпадают. План — тот же, задача 12; проверки — `frontend/src/views/pm/__tests__/MetricsView.spec.js`, разбор «MetricsView — пустые данные».

**Дата:** 2026-08-14 · **Источник:** самопроверка задачи 13 того же плана

`frontend/src/views/pm/MetricsView.vue` не различает «нет данных» и «данные нулевые». Без завершённых спринтов столбчатый график рисует пустую сетку, без назначенных задач таблица загрузки показывает одну шапку, при полном отсутствии задач накопительная диаграмма строится с нулём наборов. Падений нет, пояснения тоже. Единственное живое исключение — сгорание: при отсутствии активного спринта показывается сообщение.

**Решение:** пустое состояние на каждый блок (шаблон `#empty` у таблицы, подпись поверх графиков).

**Приоритет:** низкий, косметика.

---

## TD-052: портал фонда не пользуется portal-kit, шесть репозиториев дублируют одно и то же

**Дата:** 2026-08-14 · **Источник:** четыре аудита 13.08.2026, свод в `docs/superpowers/plans/2026-08-13-kit-reuse-audit.md`

Библиотека `portal-kit` заведена ради переиспользования заготовок между порталами разных воркспейсов. Замерено: обойдено 105 воркспейсов, портальный конфиг у 40, модуль `custom_code` — у 10 (8 различных кодовых баз). **Прикладных потребителей `@kit` два — `fund_demo/fund-app` и `soric_demo/pilot-ui`, и оба ввозят один экспорт из 39: `AiPanel`.** Остальные 38 живут только в двух стендах самой библиотеки.

При этом **шесть репозиториев, ни разу её не подключавших**, пишут у себя то же самое: разбор ссылки (6), знак отсутствия (6), пустое состояние (6), печать даты (6), плашка (7), печать суммы (5), доступ к полю (5), таблица (5), постраничное чтение (3). Реализации разошлись по поведению: `stripRef` в `uav/main.vue` отдаёт число, `sppr/main.vue` на том же входе — строку.

Внутри самого фонда до 13.08.2026 разбор числа существовал в шести редакциях, расходившихся на трёх входах (пустое поле, десятичная запятая, ссылка). Сведён к одной; `money`/`mln`/`pct` по-прежнему идут мимо неё через голый `Number()`, поэтому `money('3,5')` даёт знак отсутствия.

**Что мешает перейти:**

1. **Смыслы совпадающих имён разные.** Фондовый `NO_VALUE` — прочерк «нет данных», китовый — `·` «нормы нет»; фондовый `NO_DATA` — строка, китовый — объект (наивная замена даст `[object Object]` в 92 местах); тон `none` у фонда значит «не измерено», `neutral` у кита — «спокойно», и 64 плашки статуса начнут утверждать спокойствие.
2. **Радиус поражения.** `money` синхронна и зовётся из шаблонов, значит ввозить `@kit` в `fund-utils.js` придётся статически, а его ввозят все 40 разделов. Сегодня отказ библиотеки убивает одну панель чата; после — портал целиком. Версия при этом задаётся полем `kit` в конфиге портала, вне репозитория фонда.
3. **`DataTable` не выражает 46 % таблиц фонда** — 27 повторителей строк из 59 меняют вид строки или ловят щелчок (подсветка выбранной сущности: `linkClass` зовётся 73 раза в 31 файле из 36). Жёсткость намеренная: `ui.row` ложится на каждую строку, довод записан в самом компоненте.
4. **`EmptyState` покрывает 14 родов пустоты из 45.** Самого частого — «чего не хватает, чтобы величина посчиталась» (20 блоков) — в наборе нет вовсе.
5. **Кит не покрывает 100 имён из 110.** Регламент, эксперты ИК, субфонды, runway, учебный режим остаются в кодспейсе при любом раскладе: переход даёт смешанный код, а не чистую замену.

**Решение — порядком, а не одним движением:**

1. развести смыслы `NO_DATA` / `NO_VALUE` / тонов (без этого любой перенос примитивов меняет знаки местами);
2. перевести на китовые `numVal`, `money`, `pct`, `fmtDate`, `refText`; `money('3,5')` чинится этим же, отдельной правки не требует;
3. `EmptyState` и `StateChip` там, где своего вида не нужно, — после добавления недостающих родов;
4. `DataTable` — последним, после того как у него появится вид строки функцией от данных строки (не от её места: вид, зависящий от места, зависит от длины таблицы, и это записанный довод).

**Средний путь, если радиус поражения неприемлем:** перенести в кит только чистые функции, а в `fund-utils.js` оставить обёртки, которые их зовут. Тогда правка одна на все порталы, а отказ библиотеки ловится в одном месте и не роняет портал.

**Приоритет:** средний. Ущерб уже реализовался дважды — расхождение шести разборов числа в фонде и расхождение `stripRef` между `uav` и `sppr`.

---

## TD-053: дефекты portal-kit, найденные аудитом

**Дата:** 2026-08-14 · **Источник:** аудит устройства настройки, 13.08.2026

Закрыто в 0.9.15: слот `cell:` не отдавал объект состояния (потребитель по образцу из README печатал пустую ячейку вместо знака отсутствия); `CitationText` собирал класс мимо `uiClass`; каталог не читал `defineEmits`/`defineExpose`.

Осталось:

- **три словаря имён узлов на одни роли.** Перечень: `list` у `TeamchatPanel` и `KbBrowser`, у `AiPanel` ключа нет вовсе (`kit-ai__list` рисуется); строка перечня — `row` у `DataTable`, `item` у `KbBrowser`, у двух панелей ключа нет. Ключи добавляются, ничего не переименовывая.
- **`align` молчит на опечатке**, тогда как соседний `format` её называет (`DataTable.vue:196`). Одинаковый класс ошибки, два поведения.
- **сторож ключей `ui` срабатывает один раз** в `setup`: вычисляемая карта с опечаткой после монтирования не проверяется.
- **`compact` прячет содержание, а не только плотность** — вход и итог вызовов инструментов у `AiPanel`, описание комнаты у `TeamchatPanel`. Довода рядом нет.
- **`DataTable` не объявляет событий**: порядок сортировки наружу не уходит и не переживает перемонтирование.
- **зашитые числа без довода**: `short()` 60 знаков в двух файлах копией, `brief()` 300, `REACT_PREFETCH = 20`, `MENTION_MAX = 8`, потолок поиска 200 в двух местах одного файла.

**Приоритет:** низкий, кроме словаря имён узлов — его читает агент, и имя, которого ему негде взять, он придумывает.

---

## TD-054: `page-layout.vue` фонда не показывает иконку раздела

**Дата:** 2026-08-14 · **Источник:** аудит форм разметки

Слот `#icon` рисуется только при `live === false`, а `live` по умолчанию `true`. `captable`, `duediligence`, `gr`, `natproject` передают `#icon`, не выключив `live` — иконка не видна ни разу. Дефект записан в шапке самого `page-layout.vue` и не исправлен.

Рядом: `liveHint` не передают `capital-calls`, `gr`, `waterfall` — живая точка говорит текстом по умолчанию, который не называет, что именно раздел читает.

**Приоритет:** низкий.

---

## TD-055: процент печатается 23 способами мимо общей `pct()`

**Дата:** 2026-08-14 · **Источник:** замер после правки разбора величин

`money`, `mln` и `pct` в `fund-utils.js` с 14.08.2026 разбирают значение общим `numVal` и печатают русской записью. Но сам `pct()` зовут не везде: сплошной обход даёт **23 места в 8 разделах**, где доля печатается своим способом — `toFixed(0) + '%'`, `Math.round(v*100)`, подстановка сырого числа перед `%`.

Следствие видно на экране: в «Портфеле» стоит `12.7 %` — точка в десятичной доле посреди русского интерфейса, рядом с `50 %` и `72%` из других источников. Три записи одной величины на одном экране.

Заодно расходится и отбивка знака: `15%`, `50 %`, `12.7 %`.

**Решение:** свести к `pct()`; там, где нужна иная точность — довод `digits`, а не своя формула. Перед правкой снять слепок экранов: часть чисел изменит вид (`24.26%` → `24,3%`).

**Сделано 14.08.2026:** `money`, `mln`, `pct` разбирают величину общим `numVal` и печатают русской записью; `delta().relText` тоже — прежде одна функция печатала два числа двумя способами (`absText` через `fmtNum` с запятой, `relText` через `toFixed(1)` с точкой), и на «Портфеле» стояло `↑ 12.7 %` рядом с `50 %`. Проверено на экране: точек в долях ноль, отклонения `↑ 12,7 %`, `→ 0,0 %`, `↓ 6,8 %`.

**Осталось:** 20 мест в 6 разделах печатают процент своим способом мимо `pct()` — `finmodels.vue` 13, `gr.vue` и `twin.vue` по 2, `agent-matrix`, `captable`, `fund-twin` по одному. Часть из них — целые проценты (`toFixed(0)`), где вида это не меняет; сводить надо ради одного правила, а не ради вида.

**Приоритет:** низкий, косметика — но она про доверие к числам, а числа здесь предмет показа.

---

## TD-056: права и кэш ролей в организациях — что ветка `orgs-audit-fixes` не чинит

**Дата:** 2026-08-14 · **Источник:** доработка ветки `orgs-audit-fixes` (старшая роль при выходе, удаление всех строк членства, роли в `add_org_member`)

Пять мест найдены рядом с правкой, но выходят за её задачу: каждое меняет модель прав или срок её действия, и чинить их надо отдельно, с собственной проверкой.

- **Участник любой области организации проходит `getOrg` как `viewer`** (`backend/src/api/v2/modules/orgs/service.js:110`) — и получает перечень участников организации с почтами (`service.js:295`). Последствие: приглашённый в одну область видит весь состав организации, включая тех, с кем не пересекается.
- **Вторую строку членства с ролью `owner` можно завести через `addOrgMember`/`updateOrgMember`** (`service.js:308`, `service.js:340`; `canGrant` владельцу разрешает любую роль — `backend/src/api/v2/registry/role-utils.js:59`), а `_v2_orgs.owner_id` при этом продолжает смотреть на первого. Экран владельца прямо предлагает роль `owner` в списке выдаваемых (`frontend/src/components/MembersPage.vue:53`). Последствие: псевдовладелец получает права владельца, неудалим и неправим с экрана (`MembersPage.vue:76` и `:71` скрывают действия у любого с ролью `owner`), а сторож последнего управляющего засчитывает его вторым админом и выпускает настоящего.
- **Кэш ролей живёт 60 с и не сбрасывается при выходе из организации, удалении участника и отвязке области** (`registry/role-utils.js:75`, `clearCachedRole` там же на 103 зовётся только из `workspaces/workspace-members.js`; `orgs/service.js:430`, `service.js:488`, `service.js:278` его не трогают). Последствие: ушедший до минуты сохраняет унаследованную из организации роль в её областях.
- **Поиск области `WHERE slug = ? OR db_name = ?` без `ORDER BY`/`LIMIT` берёт `rows[0]`** (`orgs/service.js:251`, `backend/src/api/v2/modules/workspaces/service.js:238`). Последствие: если чужой `db_name` совпал со slug искомой, права проверяются на одной записи, а правка применяется к другой — порядок выдачи Postgres не обещан.
- **`LIMIT 200` на область в `getMyIssues` остаётся молчаливым** (`backend/src/api/v2/modules/orgs/pm-aggregation.js:223`) — усечение областей рядом объявляется полем `truncated`, а усечение задач внутри области ничем: 201-я задача просто не существует для читателя.

**Приоритет:** первые два — средний (модель прав), кэш ролей — средний (окно до минуты после отзыва доступа), остальные — низкий.

---

## ~~TD-057: импорт сущности графа знаний без `id` затирает чужую строку~~ ЗАКРЫТ

**Дата:** 2026-08-14 · **Источник:** разработка раздела «Онтология» портала `uav/gov` на стенде nti-new

`importEntities` берёт `id: String(e.id)` (`backend/src/api/v2/modules/swarm-memory/kag.js:302`) и кладёт строку через `ON CONFLICT (db, id) DO UPDATE` (там же, :346). Вызывающий, не передавший `id` явно, получает строку с ключом `'undefined'` — и **молча переписывает то, что лежало по этому ключу раньше**.

Замерено на себе: первое добавление сущности с портала переписало запись `uav` / `id='undefined'`, лежавшую с 25.07.2026 (уцелел только `created_at`). Дампов БД на стенде нет — **восстановить нечем**. Такая же строка-мишень есть в `udmfic` (`«Зеркало парадигмы»`, источник `manual`).

Отдельно: инструмент агента подставляет `source || 'agent'` (`backend/src/api/v2/modules/ai/agent/tools/kag.js:64`), поэтому пропущенный агентом довод уводит запись в другой источник — и поиск по нужному её не находит.

В разделе обойдено тем, что ключ задаёт портал (`portal-gov-<база36>-<случайное>`), но любой другой вызывающий по-прежнему затирает строку.

**Починка:** отвергать пустой `id` в `importEntities` либо порождать его там же, а не полагаться на вызывающего.

**Приоритет:** высокий — потеря данных без признака на экране.

**Закрыт 14.08.2026.** Платформа больше не принимает неназванный ключ: сторож `требуетКлюч`
проверяет пачку до построения векторов в `importEntities`, `importOntology` и обоих концах
`importRelations` (`backend/src/api/v2/modules/swarm-memory/kag.js`), доказательство —
`swarm-memory/__tests__/kag-import-id.test.js` (8 проверок, краснота показана до правки).
Чтобы новое требование не сломало агентов, ключ выводится из имени на границе с моделью:
`kagImportEntitiesTool` строит `<источник>-<имя>` и разводит совпадения внутри одного вызова
(`ai/agent/tools/kag.js`, проверка `ai/agent/__tests__/kag-import-derived-id.test.js`). Описание
инструмента в `TOOL_DEFS` приведено в соответствие: прежнее «если не указан — сгенерируется»
было ложным ровно до этой правки.

Две живые строки с ключом `undefined` переписаны на осмысленные ключи — `manual-зеркало-парадигмы`
(`udmfic`) и `portal-gov-реестр-допущенных-бортов-бас` (`uav`); связей, упиравшихся в них, не было,
повторная проверка даёт ноль негодных ключей. **Содержимое строки `uav`, затёртое 14.08.2026,
не восстановлено — восстанавливать нечем.** На стенде nti-new правка перенесена заплаткой по
якорям (файлы графа там расходятся с мастером на 144 и 11 строк — своя работа стенда) и ждёт
перезапуска процесса.

---

## ~~TD-058: два дефекта портального стенда nti-new, найденные при разборе `uav/gov`~~ ЗАКРЫТ

**Дата:** 2026-08-14 · **Источник:** аудит раздела `/uav/portal/gov`

- **`PATCH /api/settings/:id` писал в EAV без проверки прав** — маршрут портала не требовал ни входа, ни гранта: по номеру объекта и номеру реквизита правилось что угодно в любом воркспейсе. Маршрут существовал ТОЛЬКО на стенде, в `master` его не было.
- **`POST /api/decisions` терялся при выкладке.** Уточнение по замеру 14.08.2026: обработчик был заведён прямо на стенде (коммит `b5c048616` от 26.07.2026), в `master` его не было никогда, и выкладка положила поверх версию из мастера — обработчик исчез. Из-за этого отраслевые отчёты раздела `uav/gov` хранятся обычными EAV-таблицами (`129631`, `129632`), а не `_v2_dc_decisions`.

**Закрыт 14.08.2026.**

`PATCH /api/settings/:id` снят со стенда совсем (коммит стенда `c67c7c19b`) — это сильнее, чем
обложить правами то, чем никто не пользуется. Что его не зовут, доказано перебором, а не
предположением: обход всех восьми репозиториев кодспейса (не шести, как считалось), исходники
Nuxt-портала, собранный фронт и 36 конфигов порталов — ноль упоминаний; отзывчивость каждого
поиска проверена образцом, который он обязан находить.

`POST /decisions` возвращён **в `master`** (коммит `fb025e7f3`) со сторожем
`portal/__tests__/decisions-proxy-post.test.js` — чтобы следующая выкладка его не стёрла.
При переносе на стенд выяснилось, что там живёт ВТОРОЙ обработчик того же рода: `PATCH /decisions/:id`,
которого нет ни в одном коммите репозитория (`git log --all -S` пусто) — то есть копирование мастера
поверх повторило бы ровно ту потерю, которую чинили. Он тоже перенесён в `master` (`67a039162`),
после чего сводный файл выложен на стенд и закреплён коммитом `d28b7b316`: `git diff HEAD` по
этому файлу впервые пуст, sha256 обеих сторон совпадают.

**Правка на стенде лежит на диске, но не в памяти процесса**: `nti-backend` поднят 05.08 с
`watch: false`. До перезапуска старые маршруты живы.

**Класс, а не случай.** Оба потерянных обработчика — правки, которые жили только на стенде.
Тем же способом `portal/router.js` там разошёлся с историей на ~24 куска. Пока правка не лежит
в `master`, выкладка стирает её молча; починка — не «не забыть скопировать», а «править в
репозитории, а не на сервере».


---

## TD-059: чтение настроек портала без входа — `GET /api/settings/by-type/:typeId`

**Дата:** 2026-08-14 · **Источник:** снятие соседнего маршрута записи (TD-058)

Рядом со снятым `PATCH /api/settings/:id` на стенде nti-new живёт
`GET /:db/portal/api/settings/by-type/:typeId` (`backend/src/api/v2/modules/portal/router.js:4096`)
с собственным комментарием `no auth/grant required`: отдаёт значения EAV указанного типа любому,
кто знает номер типа, без входа и без гранта. Маршрута нет в `master` — он, как и снятый сосед,
заведён прямо на стенде.

Отличие от TD-058: это чтение, а не запись, и кто его зовёт — **не проверено**. Прежде чем
снимать, надо повторить перебор (репозитории кодспейса, исходники портала, собранный фронт,
конфиги) — у соседа он дал ноль, но здесь замера нет.

**Приоритет:** средний — утечка содержимого настроек воркспейса анониму.

---

## ~~TD-060: конверт ответа `/auth/me` уезжал в `user` целиком — портал считал гостя вошедшим~~ ЗАКРЫТ

**Дата:** 2026-08-14 · **Источник:** браузерная проверка портала

`ensureUser()` в `portal/composables/usePortalDataLayer.js` клал в `user` **всё тело ответа**:
`user.value = await res.json()`. Маршрут `GET /:db/portal/api/auth/me` анониму отвечает не отказом,
а успехом с пустой записью — `200 {"ok":true,"data":null}`
(`backend/src/api/v2/modules/portal/router.js:1083`). Конверт сам по себе непустой объект, поэтому
`api.isAuthenticated` (`computed(() => user.value != null)`, `useCustomCodeApi.js:439`) становился
**истинным для анонима**.

Круг поражения — пользовательский код порталов (`custom_code`): признак раздаётся ему как
`props.api.isAuthenticated`, а описан в `portal/docs/custom-code.md:287`, `portal/docs/composables.md:20`
и в подсказке, по которой агент пишет такой код (`backend/src/api/v2/modules/portal/chat/agents.js:98-99`).
Компонент, отпирающий раздел по этому признаку, показывал бы его гостю. Данных чужого пользователя
при этом не отдаётся: в `data` лежит `null`, а сами запросы уходят с портальной кукой и получают
анонимный ответ, — то есть это ложное отпирание показа, а не разглашение чужой записи.
Оболочка Nuxt не затронута: у неё свой `isAuthenticated` в `usePortalAuth.js:28`, он читает
`res?.data ?? {}` и сверяет `clientObjectId`. `portal-kit` тоже разворачивает конверт сам
(`TeamchatPanel.vue:462`).

**Закрыт 14.08.2026.** Конверт разворачивается тем же правилом, что и остальной слой
(`.then(r => r.data)` в `useCustomCodeApi.js`): в `usePortalDataLayer.js` заведён `unwrapEnvelope(body)` —
тело с полем `ok` несёт запись ТОЛЬКО в `data`, тело без `ok` конвертом не является. Сторож —
`portal/test/data-layer-user.test.js` (аноним — ложь, вошедший — истина и сама запись, отказ сети —
ложь); краснота показана до правки: 3 из 4 проверок падали.

Смежное, **не чинено**: `useCollection` / `useRecord` / `useReport` / `useDocuments` по-прежнему
отдают пользовательскому коду сырой конверт (`.then(r => r.json())` без `.data`), в отличие от
`useSchema` / `useDocument` / `useTimeseries` / `useRelated` и всех записывающих вызовов. Приведение
их к одному виду ломает уже написанные компоненты, поэтому делать это надо отдельной работой
с обходом кодспейс-репозиториев. Там же: пример в `portal/docs/custom-code.md:275`
(`const { data } = props.api.useCollection(...)`) неверен — эти вызовы возвращают `ref`, а не объект.

---

## ~~TD-061: `refreshUser()` не перечитывал пользователя~~ ЗАКРЫТ

**Дата:** 2026-08-14 · **Источник:** браузерная проверка портала

`api.refreshUser()` (`portal/composables/useCustomCodeApi.js`, до правки строка 440, сейчас 442) звал `ensureUser()`, а тот выходил
по флагу `userLoaded` первой же строкой. То есть единственный документированный способ обновить
данные клиента (`portal/docs/custom-code.md:288` — «Обновить данные клиента») после первого чтения
не делал ничего: вошедший в портал пользователь оставался для пользовательского кода прежним —
гостем — до перезагрузки страницы.

**Закрыт 14.08.2026.** `ensureUser({ force })` перечитывает по требованию, `refreshUser()` зовёт его
с `force: true`. Сторож — та же `portal/test/data-layer-user.test.js`, проверка «`refreshUser()`
действительно перечитывает пользователя после входа». Краснота показана **отдельно от TD-060**:
после починки конверта и до починки `refreshUser` проверка падала на `expected 1 to be 2` — второго
запроса не было.

---

## TD-062: `api.getUser()` не запускает чтение — ref остаётся пустым навсегда

**Дата:** 2026-08-14 · **Источник:** разбор кеша пользователя (TD-060/TD-061)

`getUser() { return user; }` (`portal/composables/useCustomCodeApi.js:438`) отдаёт ref, но чтения не
начинает; `user` меняет только `ensureUser()`, а его зовёт **единственное** место — `refreshUser()`.
Компонент, написанный по документации (`portal/docs/custom-code.md:286` — «Реактивный ref с записью
клиента»), получает `null` навсегда, если сам не позовёт `refreshUser()` первым. Так написан, например,
`props.api.getUser()?.value || {}` в плане фонда
(`docs/superpowers/plans/2026-08-05-fund-ic-events-twin-implementation.md:1062`).

Отсюда же следствие для TD-060: ложное «вошёл» достижимо только после вызова `refreshUser()` —
до него признак ложен просто потому, что ничего не прочитано.

**Починка:** либо `getUser()` запускает `ensureUser()` (и тогда нужна склейка одновременных вызовов —
сейчас два компонента на странице дадут два запроса), либо документация перестаёт обещать
самозагрузку. Первое честнее: имя обещает запись, а не пустой ref.

**Приоритет:** средний — молчаливая пустота вместо данных, на экране неотличима от «клиент без полей».

---

## TD-063: выложенный портал отстаёт от мастера — нет входа для сотрудников

**Дата:** 2026-08-14 · **Источник:** браузерная проверка `https://ai2o.online/promprib/portal/auth`

В мастере кнопка есть: `portal/pages/[db]/portal/auth.vue:189` —
`<button class="link-btn" @click="error = ''; mode = 'staff'">Вход для сотрудников</button>`,
и она открывает форму «почта + пароль», которая шлёт `POST /portal/api/auth/staff/login`
(там же, `:154`). На живой странице кнопки нет, при том что серверная ручка работает — значит
выложенная сборка портала старше мастера.

Не чинено намеренно: это выкладка, а не код. **Проверять надо содержимым, а не кодом ответа** —
заглушка одностраничного приложения отвечает 200 на любой адрес
(`.claude/rules/gotchas.md`, `deploy/200-не-доказательство`).

**Приоритет:** средний — сотрудники не могут войти с портала, хотя вход существует; и отдельно
это признак того, что расхождение выложенного с мастером ничем не стережётся.

---

## TD-064: инструменты `editor_*` ничего не сохраняют — документ пишется только в открытой вкладке

**Дата:** 2026-08-14 · **Источник:** сборка витрины «Профиль завода» в воркспейсе `promprib`

`backend/src/api/v2/modules/ai/agent/tool-executor.js:873-916` — восемь инструментов группы
`docs` возвращают **только команду редактору** и ничего не пишут:

```js
case "editor_insert_heading":
  return JSON.stringify({ editorOps: [{ op: "insert_heading", args }] });
```

Так устроены `editor_insert_text:873`, `_str_replace:876`, `_insert_heading:879`,
`_insert_callout:882`, `_insert_table:885`, `_insert_list:888`, `_insert_mermaid:891`,
`editor_append_section:915`. Команды исполняет **браузер** — `frontend/src/composables/useEditorOps.js`,
то есть открытая вкладка редактора с этим документом. Серверной ветки записи нет ни у одного
из них. `editor_clear_and_write:894-913` вдобавок уходит в HITL и после подтверждения тоже
пишет не сам, а отдаёт `editorOps`.

Проявление: агент (в том числе через MCP, где никакого редактора в принципе нет) вызывает
`editor_insert_heading`, получает разобранный JSON без признака отказа — и документ остаётся
пустым. Ни один из восьми ответов не содержит поля, по которому вызывающий мог бы понять,
что запись не состоялась. Инструмент, названный «вставить заголовок», молча не вставляет
заголовок.

**Обход, которым воспользовались в этой сессии:** `create_document` + `POST /:db/documents/:docId/sync-delta`
(`documents/router.js:481`) — это настоящий серверный путь записи блоков.

**Починка:** либо у `editor_*` появляется серверная ветка (та же, что у `sync-delta`), и
`editorOps` остаётся лишь оптимизацией для открытой вкладки, либо инструменты честно
объявляются требующими открытого редактора и отказывают, когда его нет. Второе дешевле,
первое полезнее: без него агент не может собрать документ вообще.

**Приоритет:** высокий — «успех» вместо записи, ровно тот класс, что TD-037/038: молча
неверный результат без ошибки.

---

## TD-065: отказ инструмента приходит с кодом 200 и `ok: true` — вызывающий по коду ответа не отличит его от успеха

**Дата:** 2026-08-14 · **Источник:** попытка поставить WHERE-фильтр отчёту в `promprib` через `update_report`

`backend/src/api/v2/modules/ai/router.js:336` (и симметрично `:367`, `:1267`, `:1375`):

```js
return res.json({ ok: true, data: { error: true, code: toolErr.code || 'TOOL_ERROR', message, ... } });
```

Любое исключение инструмента отдаётся как **HTTP 200 с `ok: true`**; признак отказа лежит
внутри — `data.error`. Задумано это для модели («пусть LLM увидит подробности и поправится»,
комментарий там же), но маршрут `POST /:db/ai/tool` — единственный вход и для MCP, и для
всякого не-модельного вызывающего. Кто смотрит на код ответа или на конверт `ok`, считает
отказ успехом.

Замер: `update_report` с WHERE, содержащим подзапрос, вернул **200** и текст
`WHERE clause references foreign schema "z" — only current workspace allowed`
(бросок из `backend/src/api/v2/utils/report-engine.js:74`). Фильтр не сохранился,
вызывающий об этом не узнал. По REST-пути того же действия (`PATCH /:db/reports/:reportId`,
`reports/router.js:94`) ошибка уходит в `next(err)` и получает честный код — то есть два
пути к одному действию расходятся в том, как сообщают об отказе.

**Смежное, отдельной строкой:** правила, по которым `sanitizeWhere` пропускает подзапрос,
нигде не описаны — ни в `TOOL_DEF` (`ai/agent/index.js:2418-2444`: сказано про `[USER]`,
`[TODAY]` и алиас `c<id>`, и ни слова про подзапросы), ни в `docs/` (`grep '{{DB}}' docs/` —
одно попадание, и то в аудите движка отчётов). А правил два, и оба неочевидны:
таблица области подставляется **только** через `{{DB}}` (`report-engine.js:68`), а алиас
подзапроса обязан подходить под `/^_[a-z]+$/i` (`:70`) — то есть `AS sub` запрещён, а
`AS _sub` разрешён. Без этого рабочий WHERE не написать иначе как перебором.

**Приоритет:** высокий — отказ, неотличимый от успеха, по коду ответа.

---

## TD-066: `run_script` глотает свою ошибку — следующие действия работают на пустоте

**Дата:** 2026-08-14 · **Источник:** разбор цепочки, из-за которой `if_else` уходил в «нет» (TD-075)

`backend/src/api/v2/modules/automations/execution.js:1171-1176`: исключение `executeIsolated`
ловится, пишется в `logger.warn` и **не пробрасывается**. Из-за этого
`env._script_result` (`:1151`) не появляется в окружении вовсе, а выполнение действий
продолжается — следующие действия читают несуществующую переменную.

Это первое звено цепочки со стенда: скрипт упал → `_script_result` нет → `if_else` с
выражением `[_script_result] > 0` подставил вместо неизвестного имени «0» → «0 > 0» честно
ответило «нет» → ветка «да» не выполнилась, и ни в одном месте не было видно, что что-то
сломалось. Второе звено закрыто (TD-075: невычислимое выражение теперь отказ, а не «нет»),
первое — нет.

**Сделано в этой сессии, но недостаточно:** ошибка запоминается в `ctx._scriptErrors`
(`:1176`) и прогон записывается в журнал как `error` (`:154-155`). То есть отказ теперь
**виден** — но цепочка по-прежнему доигрывает до конца на пустом окружении, а не
останавливается.

**Починка:** либо `run_script` бросает и обрывает цепочку, либо в окружение кладётся явный
признак «итога нет» (`env._script_result = null` вместо отсутствия имени), чтобы
последующее ветвление отказывало по TD-075, а не выдумывало ноль. Второе безопаснее для
уже написанных автоматизаций.

**Приоритет:** высокий — молчаливое продолжение на неверных данных.

---

## TD-067: определения колонок-ссылок протекают в выборки объектов

**Дата:** 2026-08-14 · **Источник:** сверка счётчиков таблиц воркспейса `promprib`

`backend/src/shared/sql-guards.js:130`, `objectWhere()`:

```js
return `${a}up != 0 AND COALESCE(${a}val, '') NOT LIKE '%:ALIAS=%' AND ${a}up NOT IN (SELECT id FROM ${safeDb} WHERE t=3 AND up=0)`;
```

Условие `up != 0` отсекает определения, лежащие под корнем. Но строка-определение
колонки-ссылки живёт под своей **записью-типом**, у которой `up != 0`, и потому проходит
общий фильтр насквозь — попадая в выборку как будто это объект.

Замер на `promprib` (число строк, отданных фильтром, против числа настоящих записей):
Захваты **26 против 17**, Подразделения **18 против 13**, Вопросы **66 против 65**.

Круг поражения — все, кто читает объекты общим фильтром: отчёты, представления
(`backend/src/api/v2/modules/views/service.js:601` — `a.t = ? AND ${objectWhere(eavSafeDb)}`),
нормализатор, `list_objects`, `count_objects`. Проявляется не отказом, а завышенными
счётчиками и лишними строками в выдаче — то есть неотличимо от «данных действительно
столько».

**Состояние:** починка ведётся параллельной работой. На момент этой записи
`backend/src/shared/sql-guards.js` в рабочем дереве **не изменён** (`git status`), то есть
в коде дефект есть.

**Приоритет:** высокий — молча неверные числа во всех выборках сразу.

---

## TD-068: множественная ссылка теряет значения — REST-путь починен, корень нет

**Дата:** 2026-08-14 · **Источник:** заполнение колонки-ссылки с несколькими значениями в `promprib`

Два места в `backend/src/api/v2/utils/eav-ref.js`:

1. **Запись, `:149`** — `let refObjId = parseInt(value, 10);`. Значение множественной ссылки
   приходит строкой «899, 954» (так его склеивают форма, ячейка CSV и AI-инструменты после
   разрешения имён в идентификаторы). `parseInt("899, 954")` возвращает **899**, всё
   после первой запятой пропадает без единого признака.
2. **Чтение, `:223-241`** — `readEavRequisite()` для ссылки берёт `LIMIT 1` и возвращает
   одно значение. Из-за этого событие `object.requisite.changed` несёт неполное «было»:
   слушатель, сравнивающий старое с новым, видит одно значение вместо списка.

**Починено в этой сессии — только REST-путь.** `backend/src/api/v2/modules/objects/service.js:2062-2075`
(`saveRequisites`) разбивает скаляр на массив до ветвления записи и пропускает даже
одиночное значение через ветку массива — та сперва очищает колонку, поэтому сужение выбора
с двух значений до одного больше не оставляет вторую строку. Сторож —
`backend/src/api/v2/modules/objects/__tests__/multi-ref-requisites.test.js` (16 проверок:
массив, строка через запятую, строка с пробелами, одиночное значение, удаление значений,
очистка; отдельным блоком — что обычная, не множественная ссылка список принимать **не**
начала).

**Не починено:** сам `eav-ref.js`. Кто пишет мимо `saveRequisites` — AI/MCP-путь, импорт CSV,
автоматизации, — теряет значения по-прежнему. Починка корня ведётся параллельной работой;
на момент этой записи файл в рабочем дереве не изменён.

**Приоритет:** высокий — потеря данных без признака.

---

## TD-069: нет события о готовности файла — дождаться распознавания нечем

**Дата:** 2026-08-14 · **Источник:** проектирование разбора захватов в `promprib`

`grep -rn 'bus\.' backend/src/api/v2/modules/files/ backend/src/api/v2/modules/normalizer/` — **пусто**.
Ни один из двух файловых конвейеров не публикует в шину ничего: ни начала обработки, ни
конца, ни отказа. Единственное извещение об окончании — WebSocket-рассылка в браузер
(`backend/src/api/v2/modules/files/doc-processor.js:526` `broadcastUpdate`, зовётся на
`:198`, `:263`, `:324`, `:400`, `:506`).

Следствие: автоматизация не может подписаться на «файл разобран». Всё, что связано с
транскриптом и распознанным текстом, приходится строить опросом по расписанию — то есть
задержкой на период опроса и лишними прогонами вхолостую. Для полевого сбора это прямо
меняет замысел: разбор захвата с голосом или фотографией идёт не по событию, а раз в минуту.

**Починка:** `bus.emit('file.processed', { db, fileId, objectId, status })` рядом с каждым
`broadcastUpdate` — в модуле есть готовое место, потому что состояние уже вычислено; плюс
триггер автоматизаций на это событие.

**Приоритет:** средний — работает обходом, но обход стоит задержки и холостых прогонов.

---

## TD-070: из песочницы автоматизаций роль всегда `admin`

**Дата:** 2026-08-14 · **Источник:** заведение проверки прав на запись в граф знаний (TD-077)

`backend/src/api/v2/modules/automations/execution.js:1142`:

```js
const scriptUser = { ...ctx.user, role: 'admin' };
```

`run_script` подменяет роль вызывающего на `admin` безусловно. Поэтому всякая проверка
прав внутри мостов песочницы для этого хозяина ничего не проверяет: заслон
`requireAdminForKagWrite` (`automations/isolated-runner.js:110-129`) для скрипта из
автоматизации всегда пропускает, как и любой будущий заслон того же вида.

Само по себе повышение может быть намеренным (автоматизация — системный субъект), но
намерение нигде не записано, а последствие — что мосты не могут различить хозяев — точно
не намеренное. Отдельно: автоматизацию заводит роль `editor`, то есть `editor` получает
через неё исполнение с правами `admin`.

**Починка:** либо роль берётся у владельца автоматизации, либо повышение остаётся, но
объявляется явно (`role: 'admin', elevated: true, reason: 'automation'`) и мосты, которым
это важно, отличают повышение от настоящего админа.

**Приоритет:** средний — повышение прав, но в контуре, куда код кладёт только админ.

---

## ~~TD-071: портальная загрузка не ставила звук в обработку — три копии одного решения~~ ЗАКРЫТ

**Дата:** 2026-08-14 · **Источник:** голосовой ввод полевого чата `promprib`

`portal/router.js` (до правки `:2925-2945`) держал **свою копию** решения «нужна ли файлу
обработка», и в этой копии не было аудио: перечень состоял из PDF, картинок, офисных
расширений и zip. Аудио, загруженное с портала — а это и `api.uploadFile()` из
`custom_code`, — не получало `processing_status` и не транскрибировалось никогда. Общая
`queueFileProcessing()` (`files/service.js:1242`) аудио знает — копия про неё не знала.

**Закрыт 14.08.2026.** Копия удалена, маршрут зовёт общую функцию
(`portal/router.js:2924-2932`). Заодно закрыты два других входа, которые вообще ничего не
ставили в очередь: `portal/teamchat/router.js:627-632` и `teamchat/router.js:559-565` —
запись, брошенная в тему, доходит до расшифровки, а не остаётся метаданными
(это то, что аудит 14.08 числил как «голосовые и фото в teamchat инертны»).
ZIP при этом перенесён в общую функцию явно (`files/service.js:1245-1249`): без переноса
удаление копии молча прекратило бы обработку архивов с портала.

Сторож — `backend/src/api/v2/modules/portal/__tests__/upload-processing.test.js`: аудио
ставится в очередь и будит обработчик, простой текст по-прежнему читается на месте,
архив по-прежнему ставится в очередь.

---

## ~~TD-072: `getPendingFiles` не выбирал `object_id` — транскрипт не доходил до записи никогда~~ ЗАКРЫТ

**Дата:** 2026-08-14 · **Источник:** разбор того, почему транскрипт не появляется в захвате

`files/service.js:1352` (`getPendingFiles`) не выбирал колонку `object_id`. Обработчик
получал файл без привязки, а `doc-processor.js` возврат текста в запись делал под условием
`if (file.objectId)` — которое было ложным **всегда**. То есть возврат транскрипта в
запись не работал ни при каком пути загрузки: ни портальном, ни обычном.

Это второй слой той же поломки, что TD-071, и он важнее: даже почини только очередь,
транскрипт всё равно остался бы в карточке файла и не дошёл бы до колонки «Транскрипт».

**Закрыт 14.08.2026.** В выборку добавлено `object_id AS objectId` (`files/service.js:1352`).
Сторож — `backend/src/api/v2/modules/files/__tests__/service.test.js`, `describe('getPendingFiles')`:
«отдаёт объект, к которому привязан файл — без него транскрипт некуда вернуть».

---

## ~~TD-073: распознанный текст не возвращался в запись вовсе~~ ЗАКРЫТ

**Дата:** 2026-08-14 · **Источник:** фотография как источник факта в полевом сборе

Возврат добытого текста в запись-владельца существовал ровно для расшифровки звука
(`writeTranscriptToObject`, колонка с алиасом `Транскрипт`). Для OCR картинки, OCR PDF-скана
и извлечения текста из офисного документа возврата не было ни в одной из трёх веток —
текст оставался в карточке файла. Фотография шильдика станка, приложенная к записи, не
давала записи ничего.

**Закрыт 14.08.2026.** Заведена одна функция на оба вида добытого текста —
`deliverTextToObject(pool, db, file, alias, text)` (`files/doc-processor.js:569-574`),
различие свелось к алиасу колонки (`ALIAS_TRANSCRIPT`, `ALIAS_RECOGNIZED`, `:41-43`).
Вызовы поставлены во все три ветки распознавания (`:201`, `:264`, `:509`) и в ветку
расшифровки (`:327`). Колонки в типе может не быть — тогда возврат молча не делается,
как и раньше.

Сторож — `backend/src/api/v2/modules/files/__tests__/doc-processor-object-writeback.spec.js`:
текст картинки и текст PDF попадают в колонку «Распознанный текст», отсутствие колонки не
ломает обработку, webm признаётся аудио и его расшифровка попадает в «Транскрипт».

---

## ~~TD-074: пустая расшифровка Whisper выглядела успехом~~ ЗАКРЫТ

**Дата:** 2026-08-14 · **Источник:** тот же прогон голосового ввода

Пустой ответ провайдера расшифровки записывался как `processing_status: 'done'` с пустым
`extracted_text`. На экране это неотличимо от разобранного файла: карточка зелёная,
текста нет. «Обработано» переставало что-либо значить.

**Закрыт 14.08.2026.** Пустой (и состоящий из пробелов) ответ теперь `processing_status: 'skipped'`
с причиной «Расшифровка не дала текста: Whisper вернул пустой ответ»
(`files/doc-processor.js:307-317`). Сторож — тот же
`doc-processor-object-writeback.spec.js`, проверка «пустая расшифровка не выглядит успехом».

---

## ~~TD-075: `if_else` с невычислимым выражением молча уходил в «нет»~~ ЗАКРЫТ

**Дата:** 2026-08-14 · **Источник:** автоматизация разбора захватов не выполняла ветку «да»

`automations/execution.js` (до правки) звал `evalFormula(action.expr || 'false', …)`.
`evalFormula` создана для вычисляемых столбцов, где пустой ответ на любую беду уместен —
в ячейке видно прочерк. Для ветвления он губителен: `null` читается как «нет». В одно и то
же «нет» сходились три разных случая — пустое выражение, неразбираемое выражение и
**отсутствующая переменная** (подстановка `[Имя]` подставляла «0», то есть выдумывала
значение). Ветка «да» не выполнялась, ошибки не было, автоматизация числилась отработавшей.

**Закрыт 14.08.2026.** Заведена `evalCondition(expr, env, место)` (`automations/helpers.js:200-317`, сама функция — `:278`):
значение по-прежнему считает `evalFormula` — вторая своя вычислялка развела бы условие
ветвления и условие запуска по разным правилам, — но три места, где формульная машина
отвечает пустотой вместо отказа, превращены в отказ (неизвестное имя; неразбираемое
выражение; сравнение `<`/`>`/`<=`/`>=` с нечисловой строкой, где `toNum("нашлось")` давало 0).
Тем же разбором пользуется и условие запуска автоматизации (`execution.js:108-119`) —
невычислимое условие записывает прогон как `error`, а не как «не подошло».

Сторож — `backend/src/api/v2/modules/automations/__tests__/if-else-expr.test.js`,
19 проверок: вычислимое «да» и вычислимое «нет» остаются собой, а нет переменной,
неразбираемое выражение, оборванное выражение, сравнение с нечисловой строкой, неизвестная
функция, пустое выражение и деление на ноль — отказ с названным местом и самим выражением
в тексте.

Первое звено той же цепочки — `run_script`, глотающий свою ошибку, — **не закрыто**, см. TD-066.

---

## ~~TD-076: три имени моста объявлены в песочнице и не проброшены — вызов падал без имени~~ ЗАКРЫТ

**Дата:** 2026-08-14 · **Источник:** попытка позвать `kag_relations` из скрипта автоматизации

`automations/isolated-runner.js` пробрасывал мосты в изолят **поимённым перечнем**
`jail.set('__query', …)`, и в перечне не хватало трёх записей: `kag_relations`,
`listDocuments`, `getDocument` жили и в мосте, и в `BOOTSTRAP_CODE`, но в изолят не
попадали. Вызов падал как `Cannot read properties of undefined (reading 'apply')` — из
сообщения не видно ни имени, ни того, что дело в проводке, а не в скрипте.

Тот же класс уже ловили в `workspace-tools/sandbox.js`, где поимённая таблица потеряла
`moveChildren`.

**Закрыт 14.08.2026.** Проброс идёт **перебором моста**, а не перечнем: новая работа моста
уезжает в песочницу сама, забыть строку негде. Плюс сторож в самой песочнице
(`isolated-runner.js`, конец `BOOTSTRAP_CODE`): имя из `__BRIDGED`, у которого хозяин
изолята не поставил моста, подменяется функцией, бросающей `«<имя>: мост не проброшен в эту
песочницу»` — не бросок на месте, потому что хозяев изолята трое и у `workspace-tools`
набор законно зависит от объявленных способностей.

Сторож — `backend/src/api/v2/modules/automations/__tests__/isolated-runner.test.js`,
`describe('isolated-runner: проводка моста в изолят')`: поиск имён проверен на отзывчивость
(находит заведомо присутствующее, не выдумывает лишнего), затем — что **ни одно** объявленное
в bootstrap имя не остаётся непроброшенным, и что вызовы `kag_relations`, `listDocuments`,
`getDocument` из песочницы доходят до своих служб.

---

## ~~TD-077: записи в граф знаний из песочницы не существовало~~ ЗАКРЫТ

**Дата:** 2026-08-14 · **Источник:** разбор захватов должен класть сущности завода в KAG

Мост песочницы умел граф знаний **только читать** (`kag_search`, `kag_relations`). Записи не
было ни в каком виде, поэтому автоматизация разбора не могла положить в KAG ни сущности, ни
связи — оставался лишь путь через админский REST-эндпоинт, недоступный изнутри скрипта.

**Закрыт 14.08.2026.** Заведены `kag_import_entities(entities, source, version)` и
`kag_import_relations(relations, source, version)` (`automations/isolated-runner.js:419-474`).
Решения, принятые по ходу и важные для последующих правок:

- Ключ (`id`) выводится из имени **тем же** `kagImportEntitiesTool`, что и у инструмента
  модели, а не второй своей выводилкой: иначе одно имя разошлось бы по двум строкам графа
  (ср. TD-057 — строка с ключом «undefined» переписывает чужую запись).
- Отказ инструмента приходит полем `error`, а не броском; здесь он превращается в бросок,
  иначе в песочницу ушёл бы «успех» с текстом ошибки внутри.
- Запись в граф — **отдельная способность** `kag_write` (`backend/src/utils/sandbox-profiles.js:49-59`),
  а не часть `write`: свалив их вместе, мы бы молча выдали запись в граф каждому уже
  объявленному инструменту с `write`. Тир — `TIER_MEDIUM`, тот же, что у одноимённых
  инструментов модели (`workspace-tools/bridges.js:19-22`).
- Проверка роли `admin` на записи (`isolated-runner.js:110-129`): `importEntities`/`importRelations`
  не принимают пользователя вообще и не проверяют ничего — вся защита живёт на маршрутах
  `POST /:db/swarm-memory/kag/import-*` под `requireRole('admin')`. Мост — второй вход в ту
  же запись. У чтения такой проверки нет намеренно: на маршруте чтения `requireRole('viewer')`.
  **Оговорка:** для `run_script` эта проверка бесполезна, потому что роль там подменяется на
  `admin` — см. TD-070.
- Счётчик — общий `mutation` (`sandbox-profiles.js:92-95`): отдельный бюджет просто перенёс
  бы разгонный цикл сюда.

Сторож — `isolated-runner.test.js`, `describe('createBridge: запись в граф знаний')`,
10 проверок: запись сущности и связи доходят до KAG; отказ инструмента приходит ошибкой, а
не «успехом»; пишется в тот же срез (`kagTags`), из которого идёт чтение; роль ниже `admin`
получает отказ, а `owner` пишет (сверка по старшинству, не по равенству); обе записи бьют по
счётчику `mutation`; способность отдельна от `write`; тир записи выше тира чтения.

**На проде этого нет** — правка лежит незакоммиченной в рабочем дереве.

---

## TD-078: двоеточие в имени колонки не переживает чтение

**Дата:** 2026-08-18 · **Источник:** заведение колонок скидки и доставки в «Усадьбе» (`usadba-3`, таблица 311)

Имя колонки хранится в EAV как `:ALIAS=Имя:`, а все читатели разбирают его нежадной
регуляркой `/:ALIAS=(.*?):/`. Двоеточие внутри имени обрывает захват на первом же
внутреннем двоеточии, и остаток имени вываливается наружу как «хвост».

Замерено: колонка, заведённая как `UDS: доставка`, хранится верно
(`:ALIAS=UDS: доставка:`, проверено `SELECT val`), но `get_table_schema` отдаёт её как
`UDS доставка:`.

Читателей десяток, и правило у всех одно, поэтому баг не локален:

| Где | Строка |
| --- | --- |
| схема | `modules/schema/service.js:62` |
| плоские представления | `modules/schema/flat-views.js:73`, `:97` |
| отчёты | `modules/reports/service.js:47`, `:510` |
| формы | `modules/forms/service.js:28` |
| объекты (REST) | `modules/objects/service.js:2026` |
| объекты (путь ИИ и MCP) | `modules/ai/agent/tools/objects.js:45` |
| командный чат | `modules/teamchat/helpers.js:196` |
| корневой индекс | `index.js:659` — здесь ещё строже: `/:ALIAS=([^:]*):/ ` |

**Следствие второго порядка — переименование дописывает алиас вместо замены.**
`updateColumn` (`modules/schema/service.js:1136-1156`) читает текущее значение тем же
`parseModifiers`, получает обрезанные `name` и `alias`, и `buildModifiers` склеивает их
заново. Переименование `UDS: доставка` → `UDS доставка` дало
`:ALIAS=UDS доставка: доставка:` и на экране «UDS доставкадоставка:».
Это же объясняет давнюю колонку «Сумма (старая)Сумма (старая)» (`usadba-3`, reqId 340).

**Обход, применённый сейчас:** двоеточий в именах нет, переименование сделано прямым
`UPDATE ... SET val = ':ALIAS=Имя:'` с последующим `rebuild_flat_views` — событие
`schema.column.updated` при правке SQL не возникает, а плоские представления живут по нему
(`modules/schema/flat-views.js:211-227`).

**Как чинить по-настоящему:** запретить двоеточие в имени на записи (валидация в
`addColumn`/`updateColumn`) либо экранировать его в хранимом значении и снять экранирование
во всех перечисленных читателях. Первое дешевле и честнее: имя с двоеточием сейчас
необратимо теряется при первом же переименовании.

---

## TD-079: препроцессор формул съедает квадратные скобки внутри строковых литералов

**Дата:** 2026-08-18 · **Источник:** написание проверки на вычисляемые колонки «Усадьбы»

`evalFormula` (`backend/src/api/v2/utils/formula-engine.js:413`) до токенизации подставляет
`[Имя]` значениями из окружения:

```js
const processed = expr.replace(/\[([^\]]+)\]/g, (_, varName) => { … return '0'; });
```

Замена идёт по сырому тексту, о строковых литералах препроцессор не знает. Поэтому любые
квадратные скобки внутри кавычек считаются именем переменной, а отсутствующая переменная
подставляется как `'0'`.

Воспроизведение (прямой вызов `evalFormula`):

```
CONCAT("[","]")   => "0"      ← должно быть "[]"
CONCAT("[","b")   => "[b"     ← одна скобка не образует пары, всё цело
CONCAT("a","]")   => "a]"
"["               => "["
```

То есть ломается только пара `[ … ]`, оказавшаяся в одном выражении, — а это ровно случай
формул, форматирующих строку в скобках.

**Как чинить:** подстановку делать не по сырому тексту, а с пропуском строковых литералов —
пройти выражение посимвольно, переключая флаг «внутри кавычек», и заменять `[…]` только вне
кавычек. Тот же проход уже есть в токенизаторе (`formula-engine.js:38-50`).

---

## TD-080: вычисляемые колонки не видны автоматизациям и не агрегируются в отчётах

**Дата:** 2026-08-18 · **Источник:** «Итого» в заказах «Усадьбы»

Вычисляемые колонки считаются в JS **после** SQL. Отсюда две дыры.

**Автоматизации их не видят вовсе.** `buildEnv` (`modules/automations/helpers.js:112`)
наполняет окружение только строками EAV из `loadAllRequisites`; `evalComputedReqsBatch`
вызывается в объектах, портале, отчётах и путях ИИ — в автоматизациях нет
(проверено грепом по `src/api/v2`). Следствие на проде: семь автоматизаций «Усадьбы»
(3, 4, 13, 14, 15, 16, 17) печатают `💰 {{req_340}} ₽` по колонке, которая заполнена
у 54 заказов из 159 за август, — то есть у двух третей сообщений бота сумма пустая,
и показать вместо неё вычисляемое «Итого» сейчас нечем.

**Отчёты их не агрегируют.** `report-engine.js:1163` пропускает вычисляемые колонки при
сборке SQL, `:1793` считает их после запроса по строкам выданной страницы. Значит по
вычисляемой колонке невозможны `SUM`/`GROUP BY`, `WHERE`, `HAVING` и сортировка. Пять
отчётов «Усадьбы» поэтому остаются на хранимом реквизите 340: «Выручка по месяцам» (151447),
«Продажи по источникам» (151469), «Задолженности по оплате» (5676547), «Заказы месяца»
(4498266); плюс личный кабинет клиента настроен на `totalReqId: 340`.

**Как чинить — два пути, и они не исключают друг друга.**

1. *Дёшево и точечно:* считать вычисляемые в `buildEnv` (обёрнуто в `try/catch` — таблица
   `_v2_computed_reqs` создаётся лениво, и в воркспейсах без вычисляемых колонок обычный
   `SELECT` из неё бросает) и класть в окружение как `computed_<id>`. Подстановка в шаблонах
   идёт по `/\{\{(\w+)\}\}/` (`helpers.js:345`), такой ключ она принимает; по русскому имени
   колонки обратиться нельзя. Для списков бота нужна отдельная правка: текст кнопки резолвит
   только `{{req_\d+}}` (`automations/telegram-helpers.js:197`).
2. *Дорого и правильно:* компилировать вычисляемое в SQL-выражение, как это делает легаси-CRM
   (`../crm/index.php:2231` `Compile_Report`): LOOKUP — `LEFT JOIN` (`:2538-2665`), ROLLUP —
   `SUM(...)` с авто-`GROUP BY` (`:2735-2770`, `:3086-3110`), агрегаты по детям —
   предагрегированной подтаблицей (`:3679-3706`), фильтр по агрегату — `HAVING` (`:3112-3137`),
   `LIMIT` внутри того же запроса (`:3612-3645`). Тогда фильтрация, сортировка и агрегация по
   вычисляемому остаются в БД.

Отдельно от обоих: материализация «по событию» под снапшот-семантику (зафиксированная цена,
итог на момент отгрузки). В легаси она сделана ровно так — запись вычисленного значения
обратно в EAV только при явном подтверждении (`../crm/index.php:4257`), без крона и триггеров.

---

## TD-081: через MCP нельзя записать в документ ничего, кроме плоского абзаца

**Дата:** 2026-08-19 · **Источник:** перенос описания проекта НТИ в воркспейс `promprib`

`append_block(docId, text, type)` и `update_block(docId, blockId, text)` принимают
содержимое ТОЛЬКО строкой и заворачивают её в дельту одним `insert`. Поля формата у них нет.

**Замер.** Положил в `text` готовую дельту Quill (`{"ops":[…]}`) и выставил `type: 'delta'`.
Сохранилось как ТЕКСТ внутри дельты: серверная функция портала `field-intake:docs`
вернула `content` = `{"ops":[{"insert":"\n{\"ops\":[{\"insert\":\"Описание проекта…`,
то есть JSON стал одним абзацем. `_v2_document_blocks.content_format` при этом
принимает `'delta'` (CHECK допускает `html | delta | config`) — ограничение чисто в MCP-слое.

**Следствие.** Заголовки, врезки (`callout`), таблицы (`simple-table`), списки и ссылки
через MCP не создаются. А без заголовков второго уровня портальный `Docs.vue` не строит
оглавление — документ выходит сплошной простынёй. Документ №3 воркспейса `promprib`
(67 КБ, 14 заголовков, 13 таблиц, 2 врезки) пришлось писать `INSERT`-ом прямо в
`promprib._v2_document_blocks` по SSH — то есть в обход и MCP, и REST.

**Как чинить:** завести у обоих инструментов `format` (`text` | `delta` | `markdown`).
При `delta` — проверять, что строка разбирается в `{ops:[…]}`, и класть как есть;
при `markdown` — собирать дельту на сервере. Разбор обязан падать на непригодной строке,
а не молча превращать её в абзац: сегодняшнее поведение — ровно тот класс тихой подмены,
когда вызов «успешен», а результат не тот.

---

## TD-082: `update_portal_module` объявляет мерж, а вложенные объекты заменяет целиком

**Дата:** 2026-08-19 · **Источник:** добавление третьего документа на портал `promprib`

В описании инструмента сказано «Новые значения для page.config (мержатся с существующими)».
Мерж поверхностный — на один уровень.

**Замер (прод, воркспейс `promprib`, страница `kb`).** Вызов с
`config: { bindings: { projectDeck: "doc:3" } }` сохранил соседей по `config`
(`kit`, `ref`, `file`, `repo`), но ЗАМЕНИЛ объект `bindings` целиком: из девяти ключей
(`docsFn`, `factoryProfile`, `fieldGuide`, `navHome`, `navData`, `navDocs`,
`navCompleteness`, `navSubstitution` и новый) остался один. На это время страница
«Документы» теряет и адрес серверной функции, и оба документа. Восстановлено повторным
вызовом с полным набором ключей; проверено чтением конфига и открытием страницы.

Опасность здесь не в самой замене, а в том, что она **молчит**: инструмент отвечает
«Модуль страницы обновлён», и расхождение видно только по последующему чтению конфига.
Рядом уже лежит гоча того же класса — `[portal/config-bindings]`: публикация из
portal-editor перезаписывает конфиг целиком и теряет привязки.

**Как чинить — одно из двух, но не полумерой:**
1. глубокий мерж для вложенных объектов (`bindings`, `settings`) с явным способом удалить
   ключ (`null` как значение);
2. либо оставить замену, но требовать набор целиком и отвечать отказом на частичный
   `bindings` — «передайте все ключи, иначе будет замена».

---

## TD-083: портал отдаёт `clientObjectId` удалённой записи, и пустая лента выдаётся за отсутствие данных

**Дата:** 2026-08-19 · **Источник:** аудит портала `promprib` в первый день выезда

`GET /api/v2/promprib/portal/api/auth/me` для `dmi@local.local` возвращает
`clientObjectId: 756`. Объекта #756 не существует — запись «Дмитрий (проверка)» в таблице
«Участники выезда» удалена 18.08.2026, `get_object(756)` отвечает `NOT_FOUND`.

**Механика.** `clientObjectId` вычисляется РОВНО ОДИН РАЗ — при входе, запросом
`portal_staff_resolve_client` (`portal/router.js:1246`) — и зашивается в JWT
(`signStaffPortalJwt`, `:1251`) на 30 суток (`PORTAL_STAFF_JWT_TTL`). Дальше он никогда не
перепроверяется: `me` печатает то, что лежит в токене. Запись удалили после выдачи токена —
и мёртвый номер живёт в куке ещё месяц. Тем же запросом сегодня `dmi@local.local` не
резолвится ни во что, то есть повторный вход дал бы `clientObjectId: null`, а старая кука
продолжает утверждать 756.

Рядом — вторая слабость того же запроса: он ищет почту по ЛЮБОМУ значению любого корневого
объекта (`WHERE LOWER(e.val) = LOWER(?) AND e.up IN (SELECT id … WHERE up = 1)`), без
привязки к таблице участников и к колонке «Портальный ID», и берёт `LIMIT 1` без порядка.
Сегодня в `promprib` совпадение единственное у каждой из четырёх заполненных почт
(проверено запросом), но как только та же строка окажется в поле другой таблицы —
пользователь молча привяжется не к той записи.

**Следствие, наблюдаемое на экране.** Главная страница портала запрашивает
`tables/310/objects?filter[req_751][eq]=756` (захваты этого автора) — ответ `total: 0`
при 30 захватах в таблице, — и печатает: «Нет захватов. Источник прочитан без отказов —
записей в нём не оказалось». То есть битая привязка пользователя подаётся как
подтверждённое отсутствие данных. Захват, созданный с этой учётной записи, уйдёт со
ссылкой на несуществующую запись.

**Как чинить:** при выдаче `me` проверять, что объект-клиент жив, и на мёртвый отдавать
`clientObjectId: null` с отдельным признаком (например `clientUnlinked: true`), чтобы
страница могла сказать «учётная запись не привязана к участнику», а не «записей нет».
Пустой список объявлять пустым можно только после успешного ответа ПО ЖИВОЙ привязке —
тот же довод, что в гоче `[portal/meta-kb]` про `loadTopics()`.

---

## TD-084: заголовок вкладки берётся из типа страницы, а не из `title` в конфиге портала

**Дата:** 2026-08-19 · **Источник:** аудит портала `promprib`

**Замер.** `/promprib/portal/contacts` → `document.title` «Контакты» при `title`
«Импортозамещение» в конфиге; `/promprib/portal/kb` → «База знаний» вместо «Документы».
У страниц типов `home`, `gallery`, `catalog` заголовок берётся из конфига правильно
(«Полевой сбор», «Данные», «Полнота»).

Мелочь на экране, но вкладка и история браузера — единственное, по чему на телефоне
различают открытые страницы; у портала, названного своими словами, две из пяти вкладок
подписаны чужими. Чинится в Nuxt-страницах типов `contacts` и `kb`: заголовок должен
приходить из `page.title`, а умолчание по типу оставаться только на случай пустого `title`.
## ~~TD-085: `UNIQUE (user_id, org_id, workspace_id)` не защищает членство в воркспейсе~~ ЗАКРЫТ

**Закрыт 21.08.2026.** В `registry/tables.js` добавлен шаг `ensureMembershipUnique`: под
сеансовым замком на выделенном соединении он снимает обломок неудавшейся сборки
(`indisvalid = f`), сводит повторы по паре (у пары остаётся строка со СТАРШЕЙ ролью —
выражение `roleLevelSql` из `role-utils.js`, то же, которым платформа уже отвечает на
несколько строк на пару), после чего строит `uq_membership_workspace`
(`WHERE workspace_id IS NOT NULL`) и `uq_membership_org` (`WHERE org_id IS NOT NULL`)
без остановки записи, а при отказе — обычной постройкой. Отказ обеих попыток не роняет
старт, но пишет в журнал ПОСЛЕДСТВИЕ: без указателя всякая вставка членства отвечает
42P10. Сторож — `registry/__tests__/membership-unique.test.js`.

**Две реализации сведены в одну при слиянии (21.08.2026).** Ветка `master` независимо
закрыла тот же долг шагом `ensureMembershipUniqueIndexes`: он опрашивал `pg_indexes`,
на найденном задвоении ОТКАЗЫВАЛСЯ строить указатель и уходил в лог, строил обычной
постройкой. Оставлена реализация с ветки `feat/workspace-carry-registry`: она умеет всё
то же и сверх того сводит повторы вместо того, чтобы откладывать защиту навсегда, и
снимает обломок неудавшейся сборки — ровно ту беду, из-за которой вторая реализация
отказывалась от `CONCURRENTLY`. Вместе с отброшенной реализацией снят и её сторож
`registry/__tests__/memberships-unique-ddl.test.js`: он утверждал обратное
(«`CONCURRENTLY` не используется»).

**Имена и предикаты сведены между ветками (21.08.2026).**
Обе стороны объявили ровно ту же уникальность под именами `uq_membership_workspace` /
`uq_membership_org`, и её указатели УЖЕ стоят на базе разработки. Останься на master свои
`_v2_memberships_ws_uniq` / `_v2_memberships_org_uniq` — после слияния на таблице
оказалось бы четыре указателя вместо двух. Предикаты равносильны на данных: строк, где
заполнены обе колонки, нет и быть не может — сплошной перебор со снятыми комментариями
даёт 17 мест вставки (`(user_id, org_id, role)` либо `(user_id, workspace_id, role)`, ни
одного с обеими), а шесть `UPDATE _v2_memberships` правят только `role` и `last_seen_at`;
замер живой базы — `ws-only|214`, прочих родов строк ноль. Форма `IS NOT NULL` при этом
строже: строку с обеими колонками накрыли бы ОБА указателя, а не пропустили бы оба.

**Отдельная беда, вскрытая тем же разбором: `middleware/apm-diag.js` писал членство
запросом `ON CONFLICT (user_id, workspace_id) DO NOTHING` — БЕЗ предиката.** Частичный
указатель Postgres берёт арбитром только когда предикат назван, иначе 42P10 «нет
уникального ограничения, соответствующего указанию ON CONFLICT». Вызов обёрнут в
`try/catch` с `log.warn`, поэтому членство там молча не создавалось. Приведено к форме
`ON CONFLICT (user_id, workspace_id) WHERE workspace_id IS NOT NULL DO NOTHING`. Других
указаний по парам `(user_id, workspace_id)` / `(user_id, org_id)` без предиката в дереве
нет — доказано сплошным перебором (22 совпадения `grep` по `INSERT INTO _v2_memberships`,
из них 5 внутри `__tests__` — образцы регулярок без списка колонок, остальные 17 разобраны
поштучно). Сторож — `middleware/__tests__/apm-diag-membership-pg.test.js`: настоящая база,
временная схема, тот самый литерал из `apm-diag.js`; рядом положительный образец — форма
без предиката обязана падать 42P10, иначе весь прогон ничего не значит.

**Файлы:**
- `backend/src/api/v2/registry/tables.js` — DDL `_v2_memberships`, ограничение `UNIQUE (user_id, org_id, workspace_id)`
- `backend/src/api/v2/modules/workspaces/workspace-members.js` — `listMembers` (ветка «владелец без строки членства»), `addMember`

**Что не так.** У членства в воркспейсе `org_id` всегда `NULL`, а SQL считает `NULL`
отличным от `NULL`. Значит ограничение на такие строки не срабатывает никогда, и
`ON CONFLICT DO NOTHING` в обоих местах вставки — мёртвый код: он ни разу ничего не
сделал.

**Замер, а не рассуждение.** Копия ровно этого ограничения, две одинаковые вставки:

```sql
CREATE TEMP TABLE zz_mem (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id BIGINT NOT NULL, org_id BIGINT, workspace_id BIGINT,
  UNIQUE (user_id, org_id, workspace_id));

INSERT INTO zz_mem (user_id, workspace_id) VALUES (9,77) ON CONFLICT DO NOTHING RETURNING id;  -- 1
INSERT INTO zz_mem (user_id, workspace_id) VALUES (9,77) ON CONFLICT DO NOTHING RETURNING id;  -- 2, НЕ пропущена
INSERT INTO zz_mem (user_id, org_id, workspace_id) VALUES (9,1,77) ON CONFLICT DO NOTHING RETURNING id;  -- 3
INSERT INTO zz_mem (user_id, org_id, workspace_id) VALUES (9,1,77) ON CONFLICT DO NOTHING RETURNING id;  -- INSERT 0 0
```

Вывод одинаков на PostgreSQL 18.4 (разработка) и 14.22 (прод): при пустом `org_id`
проходят обе вставки, при непустом вторая отсекается.

**Насколько это болит сегодня — границы находки.** Задвоений нет ни одного: запрос
`SELECT user_id, workspace_id, count(*) FROM _v2_memberships WHERE workspace_id IS NOT NULL
GROUP BY 1,2 HAVING count(*)>1` пуст и в разработке, и на проде (замер 20.08.2026).
Оба места вставки прикрыты явной проверкой перед вставкой: `listMembers` смотрит
`!rows.some(r => r.userId === ws.owner_id)`, `addMember` делает `SELECT` и отвечает 409.
То есть задвоение требует двух одновременных запросов, разошедшихся между проверкой и
вставкой. Это спящая гонка, а не работающий дефект — потому и техдолг, а не починка
на месте.

**Чем это кончится, если сработает.** Два участника с одной ролью в списке; `updateMember`
и `removeMember` бьют по `id` из URL и достанут только одну строку; `resolveEffectiveRole`
берёт `.find()` — первую попавшуюся, то есть роль станет зависеть от порядка строк.

**Как чинить.** `UNIQUE NULLS NOT DISTINCT` не подходит: он появился в PostgreSQL 15, а
прод на 14.22. Остаются два частичных уникальных индекса, работающих на 14:

```sql
CREATE UNIQUE INDEX CONCURRENTLY _v2_memberships_ws_uniq
  ON _v2_memberships (user_id, workspace_id) WHERE org_id IS NULL;
CREATE UNIQUE INDEX CONCURRENTLY _v2_memberships_org_uniq
  ON _v2_memberships (user_id, org_id) WHERE workspace_id IS NULL;
```

Порядок обязателен: сперва проверить, что задвоений нет (запрос выше — на момент правки
он пуст), иначе построение индекса упадёт. `CONCURRENTLY` — потому что таблица читается
в каждом запросе к `/:db`; вне транзакции и с проверкой `indisvalid` после.

**Правка от 21.08.2026 следует плану: первая попытка идёт `CONCURRENTLY`.** Транзакции в
`ensureRegistryTables` нет (каждый `execSql` уходит отдельным запросом в пул), то есть
`CONCURRENTLY` законен. Возражение против него — упавшее конкурентное построение
оставляет позади негодный указатель (`indisvalid = f`), который `IF NOT EXISTS` будет
обходить вечно, — снято на устройстве, а не на внимательности: перед сборкой
`ensureOneMembershipIndex` спрашивает у каталога состояние указателя через `to_regclass`
и `indisvalid` и сносит обломок сам, и то же делает перед второй попыткой. Вторая
попытка — обычное построение: оно идёт одной сделкой и при отказе не оставляет ничего.

Предикат отличается от написанного в плане: `WHERE workspace_id IS NOT NULL` вместо
`WHERE org_id IS NULL`. На данных это одно и то же (строк с обеими заполненными колонками
нет), но форма `IS NOT NULL` строже — появись такая строка, её накрыли бы ОБА указателя,
а не пропустили бы оба. Из-за этого `ON CONFLICT` обязано ПОВТОРЯТЬ предикат: частичный
указатель Postgres берёт арбитром, только когда предикат назван в самом указании.

Что после этого `ON CONFLICT DO NOTHING` без указания цели начнёт видеть частичный индекс
как арбитра — проверено, а не предположено:

```
CREATE UNIQUE INDEX zz2_ws_uniq ON zz2 (user_id, workspace_id) WHERE org_id IS NULL;
INSERT INTO zz2 (user_id, workspace_id) VALUES (9,77) ON CONFLICT DO NOTHING RETURNING id;  -- first=1
INSERT INTO zz2 (user_id, workspace_id) VALUES (9,77) ON CONFLICT DO NOTHING RETURNING id;  -- INSERT 0 0
rows = 1
```

Тогда явные проверки перед вставкой станут ускорением, а не единственной защитой.

**Что НЕ является частью этого долга.** `id: insertId || 0` в `listMembers` подозревали в
том, что при сработавшем конфликте владелец уезжает с `id: 0`. Это неверно ровно по
причине выше: конфликт не срабатывает, `RETURNING id` всегда возвращает строку. Проверять
это заново не нужно.

---

## TD-086: описания инструментов в `TOOL_DEFS` и в каталогах агентов разошлись — 299 пар из 430

**Файлы:**
- `backend/src/api/v2/modules/ai/agent/index.js` — `TOOL_DEFS`, `def.description`
- `backend/src/api/v2/modules/ai/agent/agents/*.js` — `tools[name].def.description` (24 файла)
- `backend/src/api/v2/modules/ai/router.js` — обработчик `GET /:db/ai/tools`, где решается, чьё описание доедет

**Что не так.** Одно и то же имя инструмента описано в двух местах, и до MCP доезжает ровно
одно из них. Обработчик `GET /:db/ai/tools` перебирает `TOOL_DEFS` ПЕРВЫМ и кладёт имя в
`seen`, а каталог агентов добавляет только `if (!seen.has(name))`. Значит для всякого
инструмента, объявленного в `TOOL_DEFS`, описание из каталога агента читает только
внутренний чат, а MCP — никогда. Разъехаться они могут молча, и ничто этого не ловит.

**Замер, а не впечатление** (прогон по всем 24 файлам каталогов, сверка `def.description`
строка к строке):

```
agentFiles: 24
pairs: 430          — инструмент описан и в TOOL_DEFS, и в каталоге агента
diverged: 299       — описания различаются
whitespaceOnly: 0   — ни одно расхождение не сводится к пробелам
distinctTools: 216  — сколько разных имён затронуто
```

**Чем это уже обошлось.** При добавлении поля `lastSeenAt` в `list_workspace_members`
правка ушла в каталог агента — то есть в то описание, которое MCP не читает. Дефект прожил
бы незамеченным: 848 проверок в `modules/ai` остались зелёными при разъехавшихся описаниях.
Поймано только вручную, разбором маршрута.

**Почему это опаснее, чем кажется.** Описание — единственное, из чего модель узнаёт состав
ответа. Отсутствующее поле опаснее пустого: модель подменяет его ближайшим похожим по
смыслу и печатает выдумку уверенно (см. гряблю `ai/отсутствующее-поле`). Устаревшее
описание работает так же — оно обещает не то, что придёт.

**Как чинить — по лестнице, а не записью в правилах.**

1. *Сделать невозможным:* убрать вторую поверхность. Каталог агента объявляет, какие
   инструменты агенту доступны, но не переописывает их — `def` берётся из `TOOL_DEFS` по
   имени. Тогда расходиться нечему. Требует прохода по 24 файлам и решения, что делать с
   инструментами, которых в `TOOL_DEFS` нет вовсе (у них каталог — единственный источник).
2. *Поймать проверкой:* сторож, требующий совпадения описаний для всех имён, объявленных в
   обоих местах. Прямо сейчас он покраснеет на 299 парах, поэтому вводить его можно только
   вместе с починкой либо со списком заранее известных расхождений, который может только
   сокращаться.

Точечный сторож на один инструмент уже стоит:
`backend/src/api/v2/modules/ai/agent/__tests__/list-members-description-sync.test.js`.
## TD-087: клонирование области включает нарочно погашенные автоматизации

**Дата:** 2026-08-19 · **Источник:** ветка `feat/template-carries-content`, перенос автоматизаций шаблоном

`workspace-clone.js:506-509` копирует автоматизации перечнем колонок
`(name, trigger_cfg, condition_cfg, actions_cfg, created_by, is_system)` — **без `active`**.
У колонки умолчание `active BOOLEAN NOT NULL DEFAULT TRUE`
(`automations/service.js:138`), поэтому автоматизация, выключенная в исходной области,
в клоне оказывается включённой.

**Цена.** Включается она вместе со своим `run_script`, а внутри скрипта — номера
ЧУЖОЙ области: у клона другие id типов, колонок и записей. Скрипт бьёт мимо и делает
это молча — несуществующий номер даёт пустую выборку, а не отказ. Автоматизация с
расписанием запускается сама, без единого нажатия.

**Чем доказано.** На пути шаблона тот же дефект был и закрыт в этой ветке:
`workspace-templates.js:1490` и `:2337` задают `active` явно
(`a.portable === false ? false : (a.active !== false)`), и вслух названо, почему —
«у колонки умолчание `true`, и без этого довода непереносимая автоматизация запускала
сломанный скрипт сама». Путь `clone_workspace` этого довода не передаёт.

**Как чинить:** добавить `active` в оба перечня `INSERT`/`SELECT`. Заодно учесть, что
разбора зашитых номеров на этом пути нет вовсе: `findHardcodedIds` клон не зовёт, так
что пометки «непереносимо» у него не будет — либо звать, либо переносить автоматизации
выключенными безусловно.

**Приоритет:** средний — тихая работа мимо цели в каждом клоне со скриптами.

---

## TD-088: удаление области оставляет дашборды и настройку портала

**Дата:** 2026-08-19 · **Источник:** уборка за прогоном `scripts/template-roundtrip-check.mjs`

`deleteWorkspace` (`workspaces/service.js:324-358`) сносит схему области целиком
(`DROP SCHEMA … CASCADE`), а затем чистит глобальные таблицы поимённым перечнем:
`_v2_grants`, `_v2_roles` (`:338`), `_v2_forms` (`:345`), `_v2_memberships`,
`_v2_invitations`, `_v2_workspaces` (`:349-354`). Две глобальные таблицы, помеченные
именем базы, в перечень не попали:

| таблица | чем помечена | где объявлена |
|---|---|---|
| `_v2_dashboards` | колонка `db` | `dashboards/service.js:7` |
| `_v2_portal_config` | `db VARCHAR(64) NOT NULL PRIMARY KEY` | `shared/pg-schema.js:343-351` |

**Цена.** Строки переживают удаление области. У `_v2_portal_config` имя базы — первичный
ключ, поэтому осиротевшая строка занимает его навсегда: пока она есть, `custom_domain`
удалённой области держит `UNIQUE`-ограничение
(`_v2_portal_config_domain_unique`, `pg-schema.js:350`), а область, которой это имя базы
будет выдано снова, прочитает чужую настройку портала вместо пустой.

**Чем доказано.** Замечено при уборке за прогоном сверки: после удаления временных
областей строки в обеих таблицах остались, добивались вручную.

**Приоритет:** средний — мусор копится молча, но упирается в занятый уникальный домен.

---

## TD-089 (признанное ограничение, не дефект): `workspace-templates.js` разросся до 2764 строк

**Дата:** 2026-08-19 · **Источник:** ветка `feat/template-carries-content`

Файл не делили **намеренно**: правка шла двенадцатью заходами, и деление посреди работы
стёрло бы историю правок — `git log --follow` по половинам не восстановит, кто и зачем
менял строку.

Естественный шов — между снятием манифеста и его применением:

| половина | что входит |
|---|---|
| снятие | `extractWorkspaceManifest` (`:380`) и её помощники: `parseIconVal` (`:47`), `stripSecrets` (`:209`), `findHardcodedIds` (`:226`), `tokenizeSqlIds` (`:314`) |
| применение | `applyTemplateToWorkspace` (`:1325`), `_seedDemoData` (`:1832`), `createWorkspaceFromTemplate` (`:2070`), `createWorkspace` (`:2619`) и помощники `encodeColVal` (`:39`), `encodeIconVal` (`:53`), `resolveConditionLocalIds` (`:64`), `resolveConnectorConfig` (`:77`), `initBaseTypes` (`:175`), `_resolveViewConfig` (`:195`), `detokenizeSqlIds` (`:342`), `regenerateModuleIds` (`:357`), `declaredGaps` (`:1106`), `filterManifestByEntities` (`:2035`) |

**Одна поправка к «общего помощника нет».** Перебор всех объявленных в файле помощников
по местам вызова показал ровно одно имя, живущее по обе стороны шва: `mapPortalIds`
(`:248`) — снятие зовёт его с `direction: 'extract'` (`:948`), применение с `'apply'`
(`:1594`, `:2440`). При делении его придётся либо оставить в одной половине и вывезти,
либо унести в общий файл. Остальные помощники ложатся в свою половину целиком.

**Что деление облегчает:** имена уже вывозятся (`export` у `findHardcodedIds`,
`tokenizeSqlIds`, `detokenizeSqlIds` — их читают тесты), так что шов механически чистый.

---

## TD-090: шесть молчащих `.catch` в посеве демо-данных и вставке документов

**Дата:** 2026-08-19 · **Источник:** ветка `feat/template-carries-content`

Полный перечень (получен обходом файла по образцам `catch(() => {})`,
`catch(() => ({}))`, `catch(() => [])`, `catch(() => null)`; иных форм в файле нет):

| строка | метка | что теряется молча |
|---|---|---|
| `workspace-templates.js:1950` | `demo_seed_multi_base` | значение многозначного базового поля посевной записи |
| `:1962` | `demo_seed_multi` | значение многозначной ссылки посевной записи |
| `:2009` | `demo_seed_ref` | ссылка посевной записи |
| `:2020` | `demo_seed_field` | значение поля посевной записи |
| `:1529` | `tpl_apply_doc` | **целый документ** — `.catch(() => ({}))`, дальше `if (!docId) continue` |
| `:2380` | `ws_tpl_insert_doc` | то же на пути создания области из шаблона |

**Цена.** Два последних тяжелее прочих: отказ вставки документа даёт пустой объект,
`docId` выходит `undefined`, и цикл переходит к следующему документу — ни блоков, ни
записи в `docIdMap`, ни строки в журнале. Развёрнутая область просто не имеет документа,
и узнать об этом можно только счётом.

**Почему остались.** Пять соседних мест того же рода в этой ветке переведены на
`logger.warn` (образец — `:1499-1501`, отказ вставки автоматизации собирается в
`automationsFailed` и попадает в ответ), эти шесть в перечень работ не входили.

**Приоритет:** средний для двух документных, низкий для четырёх посевных.

---

## TD-091: признак требования «Telegram-бот» шире своего смысла

**Дата:** 2026-08-19 · **Источник:** ветка `feat/template-carries-content`

Снятие манифеста объявляет требование по наличию **хоть одного** бота в области
(`workspace-templates.js:963-971`):

```sql
SELECT id FROM _v2_tg_bots WHERE db = ? LIMIT 1
```

Смысл же требования — «вход в портал идёт через бота, а токена шаблон не несёт».
Область, где бот шлёт только уведомления и портала нет вовсе, получит объявление,
которого не заслужила.

**Что уже сделано.** Текст причины в этой ветке смягчён до проверяемого (`:969`):
«У исходной области подключён Telegram-бот; его токен шаблон не несёт» — причина
утверждает ровно то, что установил запрос, и в комментарии (`:958-960`) названо, что
назначение бота запросом не проверяется. **Сам признак остался прежним.**

**Цена невелика и она односторонняя:** лишнее требование — это лишняя строка в
объявлении, а не отказ; пропущенного требования этот признак не даёт.

## TD-092: `findHardcodedIds` не видит номеров длиннее семи знаков

**Дата:** 2026-08-19 · **Источник:** ветка `feat/template-carries-content`

`workspace-templates.js:236` ищет зашитые номера образцом
`/(?<![\w.])(\d{2,7})(?![\w.])/g`. Номер в восемь знаков и больше не найдётся никогда:
автоматизация со скриптом уедет помеченной `portable: true` (`:890`), а значит
**развернётся включённой** (`:1490`, `:2337` гасят только `portable === false`) и будет
бить мимо чужими номерами — молча, см. TD-087.

**Порог назван вслух в самом коде** (`:229-235`), с ценой по обе стороны: снизу
однозначное число почти всегда счётчик или индекс, и ложная находка означает
**погашенную рабочую автоматизацию**, то есть ошибка дороже пропуска.

**Как чинить:** поднимать границу можно только с разбором ложных срабатываний на живых
скриптах — иначе размен идёт в дорогую сторону. Отдельный вопрос: сегодняшние номера EAV
до восьми знаков не доросли, так что дефект — про будущее, а не про сегодняшний день;
проверять это надо замером `MAX(id)` по областям, а не догадкой.

**Приоритет:** низкий сейчас, растёт вместе с номерами.

---

## TD-093: у публичного шаблона скрипт с номерами записей помечается переносимым

**Дата:** 2026-08-19 · **Источник:** ветка `feat/template-carries-content`

`saveAsTemplate` ставит `includeContent: vis === 'private'`
(`workspace-templates.js:1209`), поэтому шаблон видимости `org`, `public` или `system`
не несёт ни записей, ни тел документов, ни файлов репозитория — это сделано нарочно,
чтобы содержимое области не утекало через подъём видимости.

**Побочное следствие.** Карта номеров записей строится только под тем же признаком
(`:641`, `:643` — `if (includeContent && includeData && typeIds.length)`, заполнение на
`:756`). Разбор скрипта сверяется с объединением карт
`const eavAndRecords = new Map([...realToLocal, ...recordLocalIds])` (`:769`, вызов —
`:877`). При `includeContent: false` вторая карта пуста, и **скрипт, зашивший ТОЛЬКО
номера записей, находок не даёт**: `portable: scriptIds.size === 0` (`:890`) выходит
`true`, автоматизация разворачивается включённой и бьёт мимо.

Номера типов и колонок ловятся всегда — `realToLocal` собирается до всякого
`includeContent` (`:393-398`).

**Как чинить:** снимать карту номеров записей независимо от того, едут ли сами записи —
для разбора нужны только их id, а не значения.

**Приоритет:** средний — попадает ровно в общедоступные шаблоны, то есть в те, что
разворачивают чаще всего.

---

## TD-094 (признанное ограничение, не дефект): две проверки сверки ни разу не видели красными

**Дата:** 2026-08-19 · **Источник:** ветка `feat/template-carries-content`

`scripts/template-roundtrip-check.mjs` считает одиннадцать родов сущностей (`:99-103`) и
сверяет счёт исходной области со счётом копии. У десяти из одиннадцати краснота
показана на живых расхождениях. Две проверки не показаны:

1. **Строка «отчёты»** (`:92`, `SELECT count(*) … WHERE t = 22`) — единственная из
   одиннадцати, чья способность краснеть не продемонстрирована.
2. **Ветка «ссылки портала в никуда»** (`:147-153`) ни разу не срабатывала непустой:
   проверено только, что перечень пуст, когда обязан быть пуст.

По правилу `.claude/rules/prevention.md` («проверка обязана краснеть — и это доказывают,
а не предполагают») обе не считаются работающими сторожами до прогона по нарочно
испорченному входу: снять отчёт из копии и убедиться, что строка расходится; подставить в
`_v2_portal_config` копии ссылку `report:<несуществующий>` и убедиться, что ветка
печатает её и поднимает счёт расхождений.

**Это не дефект скрипта, а незакрытая часть его приёмки.**

## TD-095: правила защиты от утечек не переносятся в клон — копия мягче источника молча

**Дата:** 2026-08-21 · **Источник:** ветка `feat/workspace-carry-registry`, аудит `docs/audits/2026-08-20-workspace-lifecycle.md`

`_v2_dlp_rules` объявлена непереносимой обоими путями (`backend/src/api/v2/registry/workspace-carry.js:870-879`,
`clone: skip`, `template: skip`), а `settings` области клон копирует целиком
(`modules/workspaces/workspace-clone.js:690-695`). Значит клон включает защиту и не даёт ей ни одного правила.

**Чем плохо снаружи.** Цепочка проверена по коду от края до края: `llm-router.js:481-491` включает проверку по
`wsSettings?.dlp?.enabled`, `contour-guard.js:198-200` при пустом перечне правил возвращает `blocked: false`.
То есть на экране настроек клона DLP горит включённым, а наружу уходит ровно то, что источник запрещал.
Отказ не наступает нигде — расхождение видно только сравнением двух воркспейсов вручную.

**Почему не сделано.** Принадлежность у таблицы названа ЧИСЛОВЫМ номером области
(`workspace_id INTEGER REFERENCES _v2_workspaces(id)`), а общий внесхемный переносчик подставляет в колонку
принадлежности ИМЯ базы — вслепую он уронил бы вставку. Номер новой области известен только после её заведения
(`workspace-clone.js:691`, фаза 3), то есть нужен свой перенос отдельной фазой. Довод записан в самом реестре и
попадает в `notCarried`, спрятан он не был.

**Цена сегодня нулевая, и это надо сказать прямо:** в базе разработки правил ноль —
`SELECT count(*) FROM _v2_dlp_rules` → `0`. Излом заряжен и не сработал.

**Приоритет:** средний — стреляет у первого же клона воркспейса с настроенным DLP.

---

## TD-096 (сведено, потеря осталась): шаблонный путь объявлен несущим тридцать таблиц, а несёт восемь

**Дата:** 2026-08-21 · **Источник:** тот же аудит, раздел A-1

`template: { mode: 'manifest' }` стоит у 30 записей реестра. Что шаблон правда читает — видно по ключам,
которые возвращает `extractWorkspaceManifest` (`modules/workspaces/workspace-templates.js:551-562`):
`types, computedReqs, views, automations, roles, portal, reports, timeseries, agents`. Таблицам реестра из них
соответствуют ЧЕТЫРЕ: `_v2_computed_reqs`, `_v2_views`, `_v2_automations`, `_v2_portal_config` (и то одна
колонка `config`). Реестр шаблонный путь не ввозит вовсе: `grep -rn workspace-carry backend/src` даёт
`workspace-carrier.js`, `workspace-clone.js`, `apm-diag.js`, `workspace-ownership.js` — и ни одного попадания
в `workspace-templates.js`. Значит `template.mode` — объявление, которое никто не исполняет.

Счёт вторым, независимым способом — по упоминанию имени таблицы в тексте шаблонного пути со снятыми
блочными комментариями и с границей слова:

```
cd backend && node --input-type=module -e '
import fs from "fs";
const m = await import("./src/api/v2/registry/workspace-carry.js");
const s = fs.readFileSync("./src/api/v2/modules/workspaces/workspace-templates.js","utf8").replace(/\/\*[\s\S]*?\*\//g,"");
const t = m.CARRY.filter(e => e.template?.mode === "manifest");
console.log("manifest:", t.length, "| имени нет в файле:", t.filter(e => !new RegExp("\\b"+e.table+"\\b").test(s)).length);
'
→ manifest: 30 | имени нет в файле: 22
```

Разница между 22 и 26 — ЧЕТЫРЕ таблицы, чьё имя в файле есть, но **только на запись**: `_v2_documents`,
`_v2_document_blocks`, `_v2_dashboards`, `_v2_column_ai_config` шаблон умеет создавать из ключей `documents`,
`dashboards`, `aiButtons`, а `extractWorkspaceManifest` этих ключей не производит никогда. Путь однонаправлен:
рукописный шаблон из `backend/src/data/templates/*.json` их посеет, снятый с живой области — нет.
(В аудите 20.08 то же место названо «восемь таблиц» — это ошибка счёта: восемь имён В ФАЙЛЕ ЕСТЬ, но
четыре из них и есть настоящие несомые. Перепроверено перебором, перечень выше полный.)

**Цена.** Строк в живых областях за 26 необъявленными по делу таблицами — **574 237**:

```sql
SELECT sum(n)::bigint FROM (
  SELECT (xpath('/row/c/text()', query_to_xml(
            format('SELECT count(*) c FROM %I.%I', n.nspname, c.relname), false, true, '')))[1]::text::bigint n
  FROM pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace
  WHERE c.relkind = 'r' AND c.relname = ANY(<имена из команды выше>)
    AND (n.nspname IN (SELECT db_name FROM _v2_workspaces) OR n.nspname = 'public')) t;
```

Крупнейшие: `_v2_autofields` 561 934, `_v2_type_meta` 10 739, `_v2_document_blocks` 1058, `_v2_documents` 169.
Оговорка честности: 98 % суммы — `_v2_autofields`, а её объявление `template: manifest` само спорно (аудит, C-1:
ключ таблицы `obj_id`, то есть строка на запись, а не на колонку). Даже без неё остаётся ~12 300 строк:
область с сотней документов, дашбордом и правилами проверки колонок, снятая в шаблон и развёрнутая обратно,
теряет всё это молча — 17 глушителей в шаблонном пути: одиннадцать `.catch(() => {})`
(`workspace-templates.js:815, 841, 863, 891, 1108, 1359, 1370, 1404, 1412, 1434, 1462`) и шесть пустых
`catch { }` (`:391, 405, 447, 463, 529, 549`); в ответе — `{id,slug,name,dbName,orgId}` без единого слова
о неперенесённом. (Счёт снят заново: в аудите 20.08 шесть из одиннадцати номеров сдвинуты на строку, а два
пустых `catch` не названы; седьмой такой `catch` на `:1651` в счёт не идёт — у него есть запасная ветка.)

**СВЕДЕНО 21.08.2026 — расхождение больше не молчит, но потеря осталась.** Сделано два хода.

1. **Объявление приведено к замеру.** `template.mode: 'manifest'` теперь стоит у ДЕСЯТИ записей, и каждая
   доказана исполнением: `_v2_views`, `_v2_automations`, `_v2_computed_reqs`, `_v2_timeseries`,
   `_v2_agent_registry`, `_v2_documents`, `_v2_document_blocks`, `_v2_git_repos`, `_v2_portal_config`,
   `_v2_dashboards`. Каждая называет РАЗДЕЛ манифеста — `template.manifestKey`. Прочие 22 переведены в
   `skip` с доводом при каждой: замысел («это настройка области, ей бы ехать») остался в первой половине
   `why` и здесь, а объявление перестало выглядеть исполняемым. Заодно исправлена ложь в обратную сторону:
   `_v2_agent_registry` и `_v2_timeseries` стояли `skip`, тогда как путь их ВЕЗЁТ.
2. **Заведён сторож** — `modules/workspaces/__tests__/template-carry-registry.test.js`. Он исполняет все три
   работы шаблонного пути (снятие манифеста и ОБА пути применения) на записывающем пуле, отвечающем непусто
   на всякий запрос, и собирает имена таблиц из отрендеренного SQL. «Едет шаблоном» значит обе половины:
   раздел наполнен снятием И таблицу пишут оба пути применения. Обратная сторона стережётся тоже: всякий
   произведённый раздел манифеста либо назван записью реестра, либо изъят с доводом. Краснота доказана
   восемью нарочными изломами.

Сторож нашёл при первом же прогоне два расхождения ДВУХ ПУТЕЙ ПРИМЕНЕНИЯ между собой, оба исправлены:
`applyTemplateToWorkspace` не восстанавливал представления вовсе (раздел `views` снятие производит всегда,
а читал его только `createWorkspaceFromTemplate`), а `createWorkspaceFromTemplate` не читал раздел
`timeseries`.

**Что осталось долгом.** 22 таблицы шаблоном по-прежнему не едут, и цена та же: область, снятая в шаблон и
развёрнутая обратно, теряет правила проверки колонок, скрытость таблиц и отчётов, инструменты области,
заготовки документов, объявления рядов, настройку сведения, настройки ИИ-колонок, правила по строкам, папки
и метки документов и всю настройку nightcall. Разница со вчерашним днём в том, что теперь это записано
объявлением `skip` с доводом, а не объявлением `manifest`, которого никто не исполняет. Возврат любой из них
в `manifest` требует РАЗДЕЛА в `extractWorkspaceManifest` и чтения его ОБОИМИ путями применения — сторож
этого потребует.

**Приоритет:** был высокий (тихая потеря); стал средний — потеря та же, но названа и стережётся.

---

## TD-097: резервная копия несёт 15,7 % области, восстановление сшивает две области, а «пробный запуск» ни с чем не соединён

**Дата:** 2026-08-21 · **Источник:** тот же аудит (пути 10 и 11), план `docs/superpowers/plans/2026-08-20-workspace-backup-restore.md`

Четыре разных изъяна одного пути, ни один не чинился.

**1. Копия — только таблица EAV.** `createBackup` (`modules/admin/service.js:55-98`) читает единственный
`SELECT id, up, t, ord, val FROM <EAV>`. Замер на боевой по объёму области `usadba_3`: 613 968 строк в 68
таблицах схемы, из них в EAV — 96 317, то есть **15,7 %**. Внесхемного имущества (портал, дашборды, формы,
граф, память агентов) копия не берёт вовсе и об этом не сообщает.

**2. Число строк в ответе завышено в 24 раза.** `rows: lastId` (`admin/service.js:135`) — это наибольший номер,
а не счёт: `SELECT max(id), count(*) FROM usadba_3.usadba_3` → `2370991 | 96317`.

**3. Восстановление сшивает дерево EAV из двух областей.** `restoreBackup` (`admin/service.js:253-368`) кладёт
строки в ту область, что стоит в адресе (`POST /:db/admin/restore`), а в самом дампе признака области нет —
формат дельта-кодированный, только `id;up;t;ord;val`. Ни `DELETE`, ни `TRUNCATE` перед вставкой нет, конфликт
номеров молча пропускается (`ON CONFLICT DO NOTHING`, `:363`). Значит копия области A, залитая в область B,
даёт не состояние копии, а смесь: совпавшие номера отброшены, несовпавшие легли под чужих родителей.
`setval` при этом не делается (`grep -rn setval src/api/v2/modules/admin/` — пусто), тогда как это делают все
соседние пути — клон, шаблон, приём слепка.

**4. Переключатель «Пробный запуск» не соединён с бэкендом.** Экран шлёт `?dryRun=true|false`
(`frontend/src/services/admin.js:26`, `views/admin/Backup.vue:33`, `:97`), а маршрут читает
`req.query.execute === 'true' || === '1'` (`modules/admin/router.js:125`). Имена разные, поэтому `execute`
ложно ВСЕГДА: восстановление с экрана не пишет ни строки ни при каком положении переключателя, а всплывающее
сообщение при выключенном пробном запуске гласит «Восстановление завершено» (`Backup.vue:53-56`). Наружу уходит
`{preview: true, sql, rows}` — признак есть, но только для того, кто читает выдачу целиком.

**5. Копии лежат внутри каталога, который сносит уборка.** Архивы пишутся в
`<integram-server>/templates/custom/<db>/backups/` (`admin/service.js:107-113`), а удаление области сносит
`<serverRoot>/templates/custom/<db>` рекурсивно (`modules/workspaces/workspace-purge.js:90-95`; корень один и тот
же — `admin/service.js:21` и `modules/workspaces/service.js:28` резолвят одинаковый путь). Удаление области
уносит и все её копии. Срока хранения и уборки у копий нет.

**Чем установлено.** Числа — чтением каталога живой базы (сумма `count(*)` по 68 таблицам схемы `usadba_3`
через `query_to_xml`, `max(id)`/`count(*)` одним запросом); расхождение имён `dryRun`/`execute` — чтением обоих
концов; путь копий — сверкой двух построителей пути.

**Почему не сделано.** План на всё перечисленное написан
(`docs/superpowers/plans/2026-08-20-workspace-backup-restore.md`, 14 задач) и не исполнен: он вводит третью ось
в реестр переноса и трогает те же файлы, что и текущая работа. Из него сделан один пункт — открытый двойник
дампа рядом с архивом убран (`admin/service.js:126-131`).

**Приоритет:** высокий — п. 3 и 4 меняют данные не так, как обещано на экране.

---

## TD-098: третья копия перечня занятых слагов — на переднем крае, и ничем не связана с серверными

**Дата:** 2026-08-21 · **Источник:** тот же аудит, B-5

Перечней занятых имён теперь три, все написаны от руки и ни один не выводится из другого:

| перечень | размер | кто читает |
|---|---|---|
| `backend/src/shared/reserved-slugs.js` | 120 | `modules/workspaces/router.js:290` |
| `RESERVED` в `modules/workspaces/service.js:82` | 35 | `slugToDbName`, то есть все три пути заведения |
| `frontend/src/data/reservedSlugs.js` | 120 | мастер заведения — `WizardStepDescribe.vue:5`, `WizardStepManual.vue:7` |

Четвёртая, вырожденная, живёт в `frontend/src/stores/workspace.js:68` — шесть имён, по которым не запускается
синхронизация; отдельный вопрос, но тоже копия.

Сегодня передний край и `shared/reserved-slugs.js` совпадают побайтно (сверка множеств: 120 против 120, разницы
в обе стороны нет — прогнано разбором обоих файлов). Сторожа, который держал бы это, нет ни одного:
`grep -rn "reservedSlugs\|reserved-slugs" backend frontend --include=*.js` даёт только сами файлы и их читателей.

**Чем плохо.** Расхождение здесь молчит в обе стороны: имя, добавленное только на сервер, мастер пропустит —
и человек получит 409 после заполнения всей формы; имя, добавленное только на передний край, мастер запретит
без всякой причины на сервере. Разошедшиеся перечни того же рода уже жили в этом коде — 120 против 35, и путь
ИИ до сих пор ходит мимо длинного (B-5 аудита).

**Как чинить:** отдавать перечень с сервера (маршрут либо поле в конфигурации переднего края), а не копировать
файл. Копию можно оставить только вместе со сторожем, который её краснит при расхождении.

**Приоритет:** низкий — сегодня совпадают, цена в будущем расхождении.

---

## TD-099: два скрипта заводят строку области мимо сторожа имени, и оба написаны под MySQL

**Дата:** 2026-08-21 · **Источник:** перебор мест `INSERT INTO _v2_workspaces`

Все три штатных пути заведения добывают имя базы через `newWorkspaceDbName`
(`modules/workspaces/service.js:160-165`), которая спрашивает каталог и отказывает, если схема с таким именем
уже есть и областью не является. Сторож текста держит перечень мест от отставания
(`modules/workspaces/__tests__/workspace-name-guard.test.js:148`). Мимо него ходят два скрипта:

- `backend/scripts/import-legacy-workspace.js:117` — проверяет только занятость slug в самом реестре
  (`:107-113`), про чужие схемы не спрашивает;
- `backend/scripts/migrate-legacy.js:519` — не проверяет и этого, а `db_name` берёт равным slug без
  `slugToDbName`.

**Оба на сегодня неисполнимы, и это часть приговора, а не оправдание.** `migrate-legacy.js:39` ввозит
`mysql2/promise` и работает через `conn.execute` — это путь MySQL, платформа на PostgreSQL.
`import-legacy-workspace.js` живёт на `execSql`, то есть на PostgreSQL, но собирает имя таблицы обратными
кавычками (`:83`) и подставляет его в `CREATE INDEX` (`:100`); PostgreSQL на этом падает, а отказ проглочен
`.catch(() => {})` (`:102`), после чего печатается «Indexes ensured». То есть скрипт молча не делает
заявленного и продолжает заводить область.

**Чем плохо.** Скрипт, который «почти работает», чинят по месту — и чинят обычно ровно то, что сломалось,
а отсутствие сторожа имени переживает починку. Цена ошибки названа в шапке `workspaces/service.js:100-104`:
имя `public` доходит до `CREATE SCHEMA`, а при откате — до `DROP SCHEMA IF EXISTS "public" CASCADE`.

**Как чинить:** либо перевести оба на `newWorkspaceDbName` и `ensureAllSatelliteTables`, либо удалить как
наследие MySQL. Смешанного состояния быть не должно.

**Приоритет:** низкий, пока скрипты неисполнимы; при первой же починке — высокий.

---

## TD-100: признак «это область» написан дважды

**Дата:** 2026-08-21 · **Источник:** ветка `feat/workspace-carry-registry`

Одно и то же решение — «схема с таким именем принадлежит области или нет» — реализовано двумя функциями
`foreignSchema`: `modules/workspaces/service.js:133-150` (через `execSql`, отвечает готовым текстом отказа и
отдельно разбирает молчание каталога) и `middleware/apm-diag.js:54-63` (через `pool.query`, молчание каталога
за отказ не считает). Признак у обеих один — наличие таблицы `"<db>"."<db>"`, и довод в шапках переписан
дословно. Второй экземпляр назван временным прямо в коде (`workspaces/service.js:129-131`).

**Чем плохо.** Расхождение проявится тишиной: правка признака в одном месте оставит второй путь (приём слепка
и `split-seq`) судить по-старому. Разница уже есть — про «каталог не ответил» знает только первая.

**Почему не сведено.** `apm-diag.js` в момент правки был в работе у соседней задачи; сведение — правка его
файла.

**Приоритет:** низкий — обе реализации сегодня отвечают одинаково на всём, кроме отказа запроса.

---

## TD-101: выдача служебного ключа сворачивает его на литерале, а проверка в этом случае отказывает

**Дата:** 2026-08-21 · **Источник:** чтение путей служебных ключей

Выдача: `process.env.SK_HMAC_SECRET || process.env.JWT_SECRET || <строковый литерал в коде>`
(`modules/workspaces/workspace-bots.js:14`). Проверка: тот же выбор БЕЗ литерала, и при пустых обеих
переменных бросает `SK_HMAC_SECRET not configured` (`middleware/service-key-auth.js:14-23`), а `serviceKeyAuth`
ловит бросок и отвечает `401 AUTH_ERROR` (`:104-113`).

**Чем плохо снаружи.** На установке без обеих переменных ключ выдаётся, показывается человеку один раз как
годный — и не проходит НИКОГДА, ни через REST, ни через Basic-авторизацию git (`modules/codespace/git-server.js`
берёт того же читателя). Ответ при этом `Invalid or revoked service key`, то есть похож на неверный ключ, а не
на ненастроенную установку. Второй изъян того же литерала: он одинаков у всех установок, где переменные не
заданы, — свёртка ключа перестаёт быть тайной.

**Чем установлено.** Чтением обоих концов; сторожа, требующего `JWT_SECRET` при подъёме процесса, в коде нет —
каждый читатель отказывает сам (`iam/service.js:19-22`, `codespace/github-crypto.js:11-12`, `calls/router.js:25-26`),
и `workspace-bots.js` — единственное место, которое вместо отказа подставляет литерал.

**Как чинить:** убрать литерал и отказывать на выдаче тем же кодом, что и проверка. Ключи, выданные литералом,
после правки перестанут проходить — их и так не проходило.

**Приоритет:** средний — тихая невыдаваемость доступа плюс общеизвестная свёртка.

---

## TD-102: `create-v2-user.js` написан под MySQL и не запускается вовсе

**Дата:** 2026-08-21 · **Источник:** тот же перебор скриптов

`backend/scripts/create-v2-user.js:104` пишет членство запросом
`INSERT INTO _v2_memberships (...) VALUES (?,?,?,NOW()) ON DUPLICATE KEY UPDATE role=?` — синтаксис MySQL,
PostgreSQL такого предложения не знает (там `ON CONFLICT ... DO UPDATE`). До запроса дело, впрочем, не доходит:
файл ввозит `createPool` из `mariadb` (`:14`), а такого пакета в зависимостях нет —

```
cd backend && node -e "import('mariadb').catch(e => console.log(e.code))"
→ ERR_MODULE_NOT_FOUND
```

**Чем плохо.** Скрипт назван в шапке как способ завести учётную запись с членством и выглядит рабочим
инструментом; человек, взявший его в руки, узнаёт правду не из документации, а из отказа ввоза. Соседний
`backend/scripts/migrate-users-to-v2.js:137-143` несёт тот же синтаксис MySQL в комментарии и запросе.

**Как чинить:** удалить как наследие MySQL либо переписать на `pg` и `ON CONFLICT`. Пока не сделано ни того,
ни другого — мёртвый код в каталоге живых скриптов.

**Приоритет:** низкий.

---

## TD-103: общая база держит сирот от прежних неполных уборок

**Дата:** 2026-08-21 · **Источник:** замер живой базы разработки 21.08.2026, 206 областей

Уборка при удалении области теперь идёт по каталогу базы и берёт всякую таблицу с колонкой принадлежности
(`registry/workspace-ownership.js`, `modules/workspaces/workspace-purge.js`). След прежней уборки, ходившей по
перечню, сам не уйдёт:

| таблица | сирот | всего | колонка |
|---|---|---|---|
| `_v2_ai_audit_log` | 2660 (20 разных имён) | 16 976 | `workspace_db` |
| `_v2_ai_pending_hitl` | 29 (17 имён) | 80 | `workspace_db` |

```sql
SELECT count(*) FROM _v2_ai_audit_log
 WHERE workspace_db IS NOT NULL
   AND workspace_db NOT IN (SELECT db_name FROM _v2_workspaces);
```

**Осторожно с `doc_chunks`: там сирот НЕТ, и это ложное срабатывание того же запроса.** 2136 строк из 2445
несут `db = '__platform_docs__'` — это корпус документации платформы
(`modules/ai/agent/tools/platform-docs.js:22`, `backend/scripts/load-docs-corpus.mjs:32`), а не остаток
удалённой области; схемы с таким именем в базе нет и не должно быть. Приборка, написанная как «снести строки,
чьё имя базы отсутствует в `_v2_workspaces`», уничтожит корпус целиком. Аудит 20.08 засчитал эти 2136 строк
в сироты — счёт исправлен здесь.

**Чем плохо.** Строки удалённых областей несут содержание запросов к ИИ и ожидающие подтверждения действия,
то есть это хранение данных исчезнувших областей без срока и без владельца. На поведение живых областей они
не влияют — отбор везде идёт по имени базы.

**Как чинить:** разовая приборка отдельным скриптом, обязательно со списком исключений для служебных имён
(`__platform_docs__` и всё, начинающееся с `__`), с показом перечня удаляемого и подтверждением — по образцу
`scripts/deploy-prod.sh`. Перечень таблиц брать из каталога тем же запросом, что и уборка, а не писать заново.

**Приоритет:** низкий — гигиена, не поведение.

---

## TD-104: слепок области вывозит таблицу данных нетронутой — вырезать там нечем

**Дата:** 2026-08-21 · **Источник:** разбор `middleware/apm-diag.js` при сверке с реестром переноса

`dumpWorkspace` (шаг 2) снимает данные области запросом

```sql
SELECT id, up, t, ord, val FROM "<db>"."<db>" WHERE TRUE ORDER BY id
```

и через вырезалку `безТайного` эта выборка НЕ проходит — в отличие от спутников схемы, имущества общей базы и
самой строки области, где вырезалка применена четырежды.

**Вырезать по ключам здесь нельзя в принципе.** В EAV значение лежит опаковой строкой в колонке `val`, а имя
реквизита стоит ЧИСЛОМ в колонке `t`: образец `SECRET_KEY_RE`, которым живёт весь остальной вывоз, ловить тут
нечего. Это не недоделка вывоза, а свойство раскладки.

**Тайное там живёт, и реестр это сам документирует.** `_v2_portal_config` объявляет
`clone.strip: { config: ['telegramTokenSource', 'telegramIntegrationId', 'telegramTokenReqId'] }`
(`registry/workspace-carry.js:620`) с доводом: «вторая дорога к тому же токену: `telegramTokenSource:"table"`
читает его из EAV, а EAV клон копирует целиком и по праву — вырезается указатель, а не данные». Для КЛОНА
«по праву» верно: копия остаётся в том же контуре доверия. Файл слепка уезжает ИЗ контура — на диск, по сети,
к тому, у кого он окажется, — и там клон режет указатель, а боевой токен бота остаётся в данных и едет с ними.

Замер живой базы разработки 21.08.2026 (206 областей, 84 портальных конфига):

```sql
SELECT count(*) FROM _v2_portal_config
 WHERE config->'notifications'->>'telegramTokenSource' = 'table';
-- → 1
```

То есть дорога не гипотетическая: одна область сегодня держит токен бота именно в данных.

**Что сделано и чего не сделано.** Сделана честность, а не вырезание: слепок заявляет вывоз в
`manifest.secrets.uncut` вместе с доводом (`apm-diag.js`, шаг 2), разбор стоит в шапке файла, а сторож
`middleware/__tests__/apm-diag.test.js` («данные области едут нетронутыми, и слепок называет это, а не
умалчивает») краснеет, если запись из `secrets.uncut` пропадёт — проверено снятием записи: 1 failed | 38 passed.
Само тайное из данных по-прежнему выезжает.

**Как чинить по-настоящему.** Опознавать реквизит не по имени ключа, а по номеру колонки: пройти определения
колонок области, отобрать те, чьё имя отвечает `SECRET_KEY_RE` либо на которые указывают известные указатели
(`telegramTokenReqId` портального конфига, `tokenReqId` автоматизаций), и вырезать строки EAV с такими `t`.
Перечень указателей уже объявлен в реестре — второй писать нельзя, он отстанет молча, как отставали G_DB и
G_WSID. Отдельно решить, что делать с приёмником: вырезанный токен означает область без бота, и об этом
приёмник обязан узнать так же, как узнаёт о `secrets.tables`.

**Приоритет:** средний — утечка структурная и молчаливая, но путь доступен только суперадмину установки.

---

## TD-105 (закрыт): `bootstrap-pg-schemas.js` — четвёртая копия DDL, и сторожа реестра до неё не дотягиваются

**Дата:** 2026-08-21 · **Источник:** сверка скрипта с реестром переноса (ADR-026, §Контекст п.4)

`backend/scripts/bootstrap-pg-schemas.js` заводит таблицы области СВОИМ ТЕКСТОМ — 33 места
`CREATE TABLE IF NOT EXISTS` (строки 46–415). Реестр `registry/workspace-carry.js` объявляет 116 записей
с `home: 'ws'`. Пересчитано исполнением 21.08.2026:

```
мест CREATE TABLE IF NOT EXISTS в скрипте: 33   (из них 32 с читаемым именем; 33-е — таблица данных через %I.%I)
записей реестра с home:'ws':               116
из них скрипт НЕ заводит:                   85
скрипт заводит в схеме области, а реестр знает общей базой: 1 (_v2_dashboards, строки 415-422)
```

**Чем плохо.** Область, заведённая скриптом, получает 31 таблицу из 116 — чуть больше четверти. Остальные
достраивает ленивая инициализация модулей при первом обращении (`registry/lazy-init.js`), то есть до первого
обращения их нет, и это состояние никак не названо. `_v2_dashboards` скрипт кладёт в схему области, а реестр
объявляет её `home: 'system', scope: 'db'` — то есть скрипт и платформа расходятся не в полноте, а в том, где
таблица вообще живёт.

**Главное — сторожа сюда не смотрят.** Файловый сторож `registry/__tests__/carry-ddl-files.test.js` обходит
ровно `['src']`, и это записано в его шапке как решение: «реестр отвечает на вопрос „что везёт клон“, а
`scripts` — разовые заготовки установки, модулем не являются и записи реестра иметь не могут». Довод верен для
записи в реестре и неверен для полноты: расхождение скрипта с реестром не ловит НИ ОДНА проверка —

```sh
grep -rn "bootstrap-pg-schemas" backend/src   # → ни одного упоминания в прогонах
```

Отсюда следствие: всякая новая таблица области объявляется в реестре под красным сторожем, а в скрипте —
никогда, и разрыв 85 растёт сам собой, молча.

**ЗАКРЫТО 21.08.2026.** Взята верхняя ступень лестницы — вторая копия убрана: тело `createSatelliteTables`
зовёт `ensureAllSideTables` по `BOOTSTRAP_MODULES` (ровно то, что решено ADR-013 «Module owns its own DDL»),
и своего DDL в скрипте не осталось ни одного места. Сторож по образцу `DDL_EXEMPT` не понадобился: сверять
нечего, перечень один.

Замер обеими редакциями, скрипт запускался ЦЕЛИКОМ. Навести его на одну область удалось не доводом (своего
довода с именем области он не читает — команда из плана этапа 6 опиралась на несуществующий разбор argv), а
подставным каталогом: `PGOPTIONS='-c search_path=<схема с одной строкой _v2_workspaces>,public`.

```
прежняя редакция → 33 таблицы в схеме области
новая редакция   → 49
```

Из прежних не заводятся две, и обе по делу: `_v2_dashboards` (скрипт клал её в СХЕМУ ОБЛАСТИ с колонкой
`name`, а платформенная лежит в общей базе и зовёт её `title` — это был двойник, а не таблица платформы) и
`_v2_trash` (заводится лениво своим модулем, в `BOOTSTRAP_MODULES` не входит намеренно). Тем же доводом сняты
три индекса EAV своим текстом — `(up)`, `(t)`, `(up, t)`: их заводит модуль `eav-indexes`, и набор у него
другой — `(up, t, ord)`, `(t, ord, id)`, триграммный по `val`; про прежнюю тройку в `lazy-init.js` прямо
сказано, что две из трёх — мёртвый груз старых скриптов ввоза.

**Приоритет:** был средний — на поведение живых областей не влиял (ленивая инициализация достраивала), но
скрипт подписан как способ завести область и делал это на четверть.

---

## TD-106: образец ключей не ловит две формы `auth` у коннекторов — `credentials` и `value`

**Дата:** 2026-08-21 · **Источник:** разбор чтения `clone.blank` слепком, попутная сверка форм `auth`

Образец ключей внутри значения — `SECRET_KEY_RE` (`registry/workspace-carry.js:141`) —

```js
/secret|token|api_?key|password|authorization|bearer/i
```

Им живёт `stripSecretValues` (`workspace-carry.js:176`): обходит значение вглубь и выбрасывает ветку, чей ключ
отвечает образцу. Формы `auth` у коннектора собирает `executeConnector`
(`backend/src/api/v2/modules/connectors/service.js:613–626`), и образец разбирает их наполовину:

| форма | где секрет | ловится |
| --- | --- | --- |
| `{type:'bearer', token}` | `token` | да |
| `{type:'basic', username, password}` | `password` | да |
| `{type:'basic', credentials}` — строка `"логин:ключ"` | `credentials` | **нет** |
| `{type:'apikey', header, value}` | `value` | **нет** |
| `{type:'google_service_account', credentials}` — ключ служебного счёта | `credentials` | **нет** |

**Сегодня не течёт, и это замерено, а не предположено.** Настройки коннектора живут в EAV
(`connectors/service.js:304` и `:335` — `eavTable(db)`), а вырезание по ключам туда не ходит вовсе: значение
лежит опаковой строкой в `val`, имя реквизита — числом в `t` (это TD-104). В стриженые колонки — те, что
объявлены `clone.strip` и `clone.renewIn`, — такие значения не попадают ни разу. Замер живой базы
разработки 21.08.2026:

```sql
-- _v2_portal_config.config: 82 узла auth, ни одного в форме коннектора
SELECT count(*), count(*) FILTER (WHERE a ? 'credentials' OR a ? 'value')
  FROM _v2_portal_config c, LATERAL jsonb_path_query(c.config, '$.**.auth') a;   -- → 82 | 0
```

`_v2_automations.actions_cfg` и `trigger_cfg` обойдены тем же `jsonb_path_query('$.**.auth')` по всем 208
схемам, где эта таблица есть, — **0 узлов**. Отзывчивость обхода доказана третьим замером, по самим данным
области: `val LIKE '%auth%' AND val LIKE '%credentials%'` по 209 таблицам EAV даёт **7 строк** (одна и та же
настройка коннектора UDS в семи областях, `{"auth":{"type":"basic","credentials":"…"}}`) — то есть ключ такой
формы в базе ЖИВОЙ, он просто лежит там, куда вырезание не ходит.

**Чем плохо.** Попадёт такое значение под `clone.strip` — и `stripSecretValues` пронесёт ключ в копию либо в
шаблон молча: ветку `auth` он оставит целиком, потому что ни `credentials`, ни `value` образцу не отвечают.
Дорога к этому короткая и обычная: колонка JSONB, объявленная `strip` (портальный конфиг и автоматизации уже
объявлены), плюс действие, хранящее настройку внешнего вызова рядом с прочей.

**Как чинить.** Дописать `credentials|value` в образец НЕЛЬЗЯ: `value` — слово общего употребления, а промах
образца ключей в сторону лишнего молчалив (значение уезжает укороченным, и находится это отказом читателя в
копии — разбор в шапке `workspace-carry.js:104–139`). Резать узел `auth` целиком тоже нельзя: замер выше
показывает 82 узла `auth` в портальном конфиге, и все до одного — карта полей входа (`{nameReqId, phoneReqId,
clientsTypeId}`) либо настройка входа; вырезанная, она оставляет копию БЕЗ ВХОДА В ПОРТАЛ
(`findOrCreateClient`, `portal/auth.js:406`). Годный ход — судить по СОСЕДУ: у узла с ключом `type` из
известного перечня форм секрет назван самим `type`, и перечень этот в коде уже есть дважды —
`connectors/service.js:613–626` и `buildAuthHeaders` (`agent-registry/service.js:122`). Значит объявить его
один раз рядом с образцом и читать обоими, а не заводить третий. Сторож — по образцу
`carry-portal-secrets.test.js`: карта полей входа уцелела, ключ формы коннектора вырезан.

**Приоритет:** низкий — сегодня ни одного попадания, но выдаст себя только утечкой.

---

## TD-107: наложение шаблона на живую область не восстанавливает роли

**Дата:** 2026-08-21 · **Источник:** сторож `template-carry-registry.test.js`, разбор двух путей применения

Раздел `roles` манифест несёт всегда (`extractWorkspaceManifest` читает `_v2_roles` по `workspace_id` с
`is_system = false`), и `createWorkspaceFromTemplate` его пишет — `INSERT INTO _v2_roles (workspace_id, name,
description, is_system) VALUES (?, ?, ?, 0)`. Второй путь применения, `applyTemplateToWorkspace`, раздела
`roles` не читает ВОВСЕ: `grep -n "manifest.roles" backend/src/api/v2/modules/workspaces/workspace-templates.js`
даёт одно попадание, и оно в теле заведения новой области.

Это то же расхождение двух путей, что было с представлениями и рядами данных (найдено и исправлено там же
21.08.2026), но починить его тем же движением нельзя: у `_v2_roles` есть уникальность имени внутри области и
уже выданные права, поэтому «дописать роль из шаблона» — не вставка, а слияние, и решение о нём отдельное.

**Чем плохо.** Шаблон, наложенный на живую область, приносит представления, автоматизации, отчёты и правила
— и не приносит ролей, под которые они писаны. Молча: в ответе `applyTemplateToWorkspace` о ролях нет ни
слова.

**Почему сторож этого не ловит.** `_v2_roles` — глобальная таблица платформы, записи в реестре переносимого
у неё нет и быть не может (`registry/tables.js`), поэтому раздел `roles` стоит в изъятии
`РАЗДЕЛЫ_НЕ_ИЗ_РЕЕСТРА` сторожа. Изъятие верно по своему доводу и не обязано отвечать за поведение.

**Приоритет:** средний — потеря видимая на экране прав, но необъявленная.

---

## TD-108: две реализации объектов и паритетные дыры обеих дверей

**Что сделано и почему остаток отдельный.** ADR-027 свёл судейство на пути
инструментов ИИ к тому же, что у REST: построчные правила в пяти функциях
`ai/agent/tools/objects.js`, автополя при создании и правке, флаг `DELETE` и
отказ при входящих ссылках у `delete_object`/`bulk_delete`, таблица
`TOOL_MIN_ROLE` на 35 имён и сторож, выводящий ожидание из REST-маршрутов.
Ниже — то, чего эта работа намеренно не касалась.

**1. Реализаций объектов по-прежнему две.** `ai/agent/tools/objects.js` и
`modules/objects/service.js` живут отдельно: своя пагинация, своя форма ответа,
своя сортировка по ссылке, свои вычисляемые колонки. Судьи теперь одни и те же,
но код разный, и правка по-прежнему может уехать в одну половину. Свести —
работа на 150-250 строк только по `list_objects`, и делать её надо после того,
как права закрыты, а не вместо.

**2. Паритетные дыры, где судьи нет НИ на одной двери** — то есть это не
расхождение путей, а дефект платформы:

- корзина (`modules/objects/trash.js`): ни один из трёх маршрутов не судит
  доступ; `getTrashItem` отдаёт все реквизиты удалённой записи,
  `restoreFromTrash` воскрешает её без единой проверки;
- обратные ссылки (`getObjectBacklinksTool`) — грант на тип не спрашивается;
- реакции на комментарии (`addReaction`/`removeReaction`) — не проверяют ничего;
- `count_objects`, `aggregate_objects`, `pivot_objects` — грант есть, построчного
  правила нет, а `pivotObjects` отдаёт содержимое полей в подписях.

**3. Разбор 28 файлов `tools/` — сделан 22.08.2026, и прежняя оценка была
завышена.** Грубый поиск давал «26 файлов без судьи»; он не раскрывал
делегирование. Перебор с раскрытием (для каждого экспорта: свой судья → вызов
служебной функции, у которой судья внутри → ничего) даёт **31 экспорт в восьми
файлах**, и **ни одного расхождения дверей среди них**:

| файл | экспортов без судьи | чем закрыто на деле |
| --- | --- | --- |
| `portal.js` | 8 | ролью `admin` в `TOOL_MIN_ROLE` (ADR-027) |
| `documents.js` | 6 | ничем — но и REST-двойник ничем: у документов нет модели грантов вовсе |
| `nightcall.js` | 4 | шлюзом модуля, выключенного по умолчанию |
| `workspace.js` | 4 | `readFileTool`/`searchFilesTool` — у файлов нет модели грантов ни на одной двери (`checkGrant` в `modules/files/` встречается 0 раз) |
| `graph.js` | 3 | ничем, как и REST |
| `platform-docs.js` | 3 | судья не нужен: читают документацию платформы, не данные воркспейса |
| `pm.js` | 2 | ничем, как и REST |
| `schema.js` | 1 | `addColumnToReport` — чтение |

**Отсюда настоящий остаток:** у файлов, документов, графа и pm нет модели прав
вообще — единственный заслон это членство в воркспейсе. Это не расхождение
дверей и не следствие ADR-027, а незакрытая область платформы; закрывать её
надо решением о модели, а не подстановкой судей по местам.

**Приоритет:** пункт 2 — закрыт 22.08.2026 (корзина, агрегаты, реакции,
обратные ссылки). Пункт 1 — средний. Пункт 3 — средний, и это вопрос модели, а
не правки.

---

## TD-109: ротация токенов Telegram-ботов после утечки #143

**Дата:** 2026-08-18 · **Источник:** закрытие утечки секретов портального конфига (issue #143)

Токены из `_v2_portal_config.config.notifications.telegramBotToken` были доступны в публичном
`/api/config` и в SSR-HTML портала. Утечка закрыта, значения — нет: ротация делается владельцем
в BotFather. Перечень задетых воркспейсов: `cd backend && node scripts/portal-secret-audit.mjs`.
Пока скрипт возвращает 1, ротация не завершена.

**Замер при вливании 22.08.2026.** На боевом `integram-new` конфигов 42, и ни в одном из них
`telegramBotToken` нет — то есть **ротировать на проде сегодня нечего, перечень пуст**. Долг
остаётся открытым по двум причинам: значение, побывавшее в публичном HTML, засвечено навсегда
(поле могли удалить руками уже после утечки), и первый же портал, где токен задан в редакторе,
снова наполнит перечень. На дев-базе скрипт возвращает 1 и называет два воркспейса
(`test_fermer`, `usadba_3`) из 84 конфигов.

---

## TD-110: `webhook_secret` Telegram-бота отдаётся инструментами MCP

**Дата:** 2026-08-18 · **Источник:** ревью закрытия утечки секретов портального конфига (issue #143)

`webhook_secret` — это `sha256(token)`, обрезанный до 32 знаков (`makeWebhookSecret` в
`backend/src/api/v2/modules/portal/service.js`). Сам по себе он необратим: токен бота из него не
достать. Но это ровно то значение, которым Telegram подписывает входящие вебхуки —
`setWebhook` передаёт его параметром `secret_token`, а обработчик `/portal/telegram/:secret`
сверяет с ним заголовок `X-Telegram-Bot-Api-Secret-Token` (`telegram-webhook-secret.js`).
**Зная его, участник воркспейса может слать поддельные «сообщения от Telegram» на вебхук бота.**

Пути наружу, проверенные по коду:

- `create_telegram_bot` — кладёт `webhook_secret` явным полем в ответ инструмента
  (`createTelegramBotTool` в `ai/agent/tools/portal.js`);
- `get_telegram_bot_status` — отдаёт `webhookInfo` как есть, а Telegram возвращает в нём
  зарегистрированный `url`, который строится с этим секретом в пути;
- `list_telegram_bots` — самый широкий: `listBots` выбирает колонку `webhook_secret` для всех
  ботов воркспейса, инструмент отдаёт строки без правки. Тир `TIER_LOW`, подтверждения нет,
  проверки роли в `tool-executor.js` тоже нет — то есть секрет доступен любому участнику,
  дошедшему до агента или MCP.

REST-маршруты `/portal/api/bots*` возвращают то же поле, но закрыты `requireRole('admin')`.

Отдельная работа: в issue #143 не входит, вырезателем секретов конфига не закрывается —
`webhook_secret` живёт в `_v2_tg_bots`, а не в `_v2_portal_config.config`.

**Дополнение 22.08.2026 — вторая дверь к тому же значению, и она закрыта.** У портала есть
СВОЯ колонка с тем же секретом: `_v2_portal_config.telegram_secret`, туда его кладёт
`POST /api/config` после регистрации вебхука. `getConfig` её не выбирает, а `RETURNING *` в
`upsertConfig` и `setActive` — выбирал, и она уезжала мимо вырезателя (тот ходит по полю `config`,
а колонка лежит рядом) в ответ `POST /api/config`, `PUT /api/config/active` и в ответ инструментов
`set_portal_config` / `update_portal_module` — у последних двух ни подтверждения, ни проверки роли
нет. Закрыто при вливании **у запроса, а не у каждого читателя**: `RETURNING *` заменён поимённым
перечнем `PUBLIC_ROW_COLUMNS`, общим с `getConfig`, — колонка больше никуда не отдаётся и заслонов
у потребителей не нужно. Сторож — `portal/__tests__/upsert-config-guards.test.js`, сверяет весь
перечень, а не отсутствие имени: `RETURNING *` имени тоже не содержит и прошёл бы такую проверку.
Прежний сторож молчал потому, что заглушка строки колонки не несла вовсе. Остаток долга —
`_v2_tg_bots.webhook_secret` в `create_telegram_bot`, `get_telegram_bot_status`,
`list_telegram_bots` — не тронут.

---

## TD-111: выключатель модуля `codespace` не годится заслоном, пока его умолчание — `false`

**Что не так.** `requireModule('codespace')` (`middleware/jwt-auth.js`) читает
`settings.modules.codespace` и закрывает дверь на `false`. Но `false` там стоит
**не по решению владельца, а по умолчанию платформы**: `DEFAULT_MODULES` в
`modules/workspaces/service.js:50` объявляет `codespace: false // disabled by
default — requires GIT_ROOT setup`, а `normalizeSettings` тут же дописывает
явные булевы всем модулям — то есть область заводится уже с `false` в поле, и
отличить «никто не включал» от «владелец выключил» по нему нечем. Оговорка про
`GIT_ROOT` на боевом стенде вдобавок неактуальна: он там задан глобально
(`GIT_ROOT=/opt/integram/git-repos`), настройки не хватает не в области.

**Замер боевого стенда 22.08.2026.** Читающий: `psql` по `_v2_workspaces`,
`_v2_portal_config` и `<db>._v2_git_repos`, затем анонимные `GET` на
`https://ai2o.online`. Ничего не изменено.

| что | результат |
| --- | --- |
| выключатель `codespace` по областям | `false` у 21, `true` у 7, ключа нет у 82 (всего 110) |
| портальных конфигов | 42 |
| модулей `custom_code` | 19 в 11 областях; у двух (`usadba_3`, `usadba_3_stagin`, порталы `active: false`) в конфиге нет ни `repo`, ни `file` — блоб-роут они не зовут |
| живых модулей, читающих блоб-роут | 17, все отвечают `200` анониму |
| из них в областях с `codespace: false` | **7** — `promprib` (5) и `soric_demo` (2) |

То есть выключатель, поставленный на портальный блоб-роут, погасил бы семь
боевых разделов из семнадцати, и ни один из владельцев этих областей ничего не
выключал. Поэтому заслон снят (`modules/portal/router.js`, маршрут
`/api/codespace/:repo/blob/:ref/*`), и та же причина уже названа у студии кода
(`/api/code/tree`, `/api/code/file`). Оба места держатся сторожами
(`blob-route-access.test.js`, `code-studio-access.test.js`), доказанно
краснеющими при возврате `requireModule`.

**Чем платим.** Расхождение дверей (ADR-027): у области с `codespace: false`
REST-дверь кодспейса `/:db/codespace` отвечает участнику с токеном `403
MODULE_DISABLED`, а портальная дверь тот же репозиторий анониму отдаёт. Сегодня
портальная закрыта другим, и этого хватает — репозиторий назван конфигом, `ref`
назван конфигом, роль видит модуль, формат в белом списке
(`portal/code-guard.js`), — но по выключателю модуля двери судят по-разному, и
это надо чинить не на портальной стороне.

**Что сделать, чтобы выключатель стал годным заслоном** (одно из двух, не оба):

1. **Развести «не настроено» и «выключено владельцем».** Сегодня оба состояния
   пишутся одним `false`. Различать их можно только третьим значением или
   отдельным полем «кто выключил»: пока значение одно, любой заслон на нём —
   выключатель вслепую. Правка затрагивает `DEFAULT_MODULES`,
   `normalizeSettings` и `requireModule`, плюс перенос уже записанных `false` у
   21 области — а перенос надо делать, зная, что у семи из них модуль на деле
   работает.
2. **Снять умолчание `false`, оставив проверку `GIT_ROOT` там, где она
   осмысленна** — при заведении репозитория, а не при заведении области.
   Тогда `codespace` встаёт в общий ряд opt-out модулей, и `false` в поле
   означает ровно решение владельца.

До этого `requireModule('codespace')` нельзя ставить ни на портальную дверь, ни
на студию кода. Проверено обратным ходом: с возвращённым сторожем прогон
`blob-route-access.test.js` падает на «modules.codespace=false — файл всё равно
отдаётся».

**Приоритет:** средний. Ущерба сегодня нет, но всякая следующая попытка
«закрыть портальную дверь выключателем модуля» повторит ту же поломку.

---

## TD-112: заслон блоб-роута держится краями, которых сам не проверяет

**Где.** `backend/src/api/v2/modules/portal/code-guard.js`,
`backend/src/api/v2/modules/portal/router.js` (маршрут
`/api/codespace/:repo/blob/:ref/{*path}`),
`backend/src/api/v2/modules/codespace/service.js`.

Три края, найденные перепроверкой заслонов issue #140. Ущерба сегодня нет ни от
одного, но каждый держится не на том, что написано в заслоне.

**1. Правило про серверные функции обходится записью пути.** Образец
`/^api\/[^/]+\.js$/i` (`code-guard.js`) пропускает `./api/fn.js`, `api//fn.js`,
`api/./fn.js` — все три `isBlobFileAllowed` признаёт годными. Не стреляет
только потому, что репозитории голые, а git на такие пути отвечает отказом.
Замер 22.08.2026 в bare-клоне (`git show <sha>:<путь>`): `api/fn.js` — отдаёт
содержимое; `./api/fn.js` — `fatal: relative path syntax can't be used outside
working tree`; `api//fn.js` и `api/./fn.js` — `fatal: path … does not exist`.
**Чем платим:** защита исходников серверных функций стоит на свойстве
хранилища, а не на заслоне. Любая нормализация пути на пути чтения — кэш,
рабочее дерево вместо bare, замена `git show` на библиотеку — открывает её
молча. Лечится нормализацией пути ДО проверок (сегмент `.`, пустой сегмент,
ведущий `./`) в самом `isBlobFileAllowed`.

**2. `..` ловит не заслон, и код отказа расходится с прочими.** В заслоне
правила на выход из дерева нет вовсе: путь доходит до
`validateFilePath` (`codespace/service.js:120`) уже внутри `readBlob`, и тот
отвечает `400 VALIDATION_ERROR`, тогда как все отказы блоб-роута — `401`/`403`.
**Чем платим:** во-первых, отличие кода само по себе отвечает на вопрос, дошёл
ли запрос до чтения; во-вторых, заслон окажется без этой проверки, как только
чтение позовут иначе — сегодня она достаётся ему по случайности маршрута, а не
по замыслу.

**3. Репозиторий из базы читается раньше проверок ref, роли и формата.**
`router.js` зовёт `codespaceService.getRepo` сразу после `REPO_NOT_IN_CONFIG`, а
`getRepo` (`codespace/service.js:404`) выполняет `ensureCodespaceTables` (DDL) и
бросает `404 NOT_FOUND`. **Чем платим:** аноним отличает «репозиторий назван
конфигом, но не заведён» (404) от «заведён, но закрыт» (401/403) — то самое
различение, ради устранения которого проверку `REPO_NOT_IN_CONFIG` и подняли
выше обращения к базе; и DDL с запросом идут на анонимный запрос до всякой
проверки роли. Лечится переносом ветки/роли/формата выше `getRepo` там, где они
от репозитория не зависят, — но ref-заслону нужен `default_branch`, поэтому
правка не механическая.

**Приоритет:** низкий по ущербу, средний по хрупкости. Пункт 1 — первым: он
единственный, где заслон уже сегодня решает не то, что написано.

---

## TD-113: браузер и заслон подставляют разные умолчания — ветку и роль

**Где.** `portal/components/modules/ModuleCustomCode.vue:24`,
`backend/src/api/v2/modules/portal/router.js` (блоб-роут),
`backend/src/api/v2/modules/portal/auth.js`.

Два места, где заслон блоб-роута считает не то же, что считает экран, и обе
цены — погасший раздел, а не утечка.

**1. Ветка по умолчанию.** Браузер просит `module.config.ref || 'main'`
жёстко; заслон разрешает `mod.config.ref || repo.default_branch || 'main'`.
На репозитории, у которого ветка по умолчанию зовётся не `main`, модуль без
явного `ref` получает `403 REF_NOT_ALLOWED` на живом разделе. Сегодня спит:
замером 22.08.2026 все боевые репозитории зовут ветку по умолчанию `main`
(тот же замер — в `.claude/rules/gotchas-portal.md`, запись `portal/staff-жёсткий-ref`).
`backend/scripts/codespace-blob-exposure.mjs` расхождение печатает отдельной
строкой. **Чем платим:** первый же репозиторий с `master` или `release` в
`default_branch` гасит раздел, и причина не видна ни на экране, ни в конфиге.
Лечится сведением умолчания в одно место — либо браузер спрашивает ветку у
конфига полностью, либо заслон принимает и `'main'`, и `default_branch`.

**2. Роль сотрудника портала.** Блоб-роут берёт роль через
`loadClientRole(..., req.portalClient.clientObjectId)`. У сотрудника
`clientObjectId` — необязательное поле токена: `signStaffPortalJwt`
(`auth.js:472`) кладёт его, только если вход нашёл запись по адресу почты
(`router.js`, запрос `portal_staff_resolve_client` — сопоставление
`LOWER(val) = LOWER(email)` по всем корневым записям), а `requireStaffPortalJwt`
ставит `clientObjectId: null` прямо. Совпадения нет или оно пришлось на запись
без `roleReqId` — `roleName` пуст, и сотрудник получает `403 ROLE_REQUIRED` на
ролевом модуле, который на экране ему виден. **Чем платим:** выстрелит на
`/portal/staff` — странице, куда роль добавят первой; сегодня ни один боевой
конфиг ключа `roles` не содержит, поэтому ветка мертва. Лечится тем же
средством, что и на экране: сотруднику роль обязана даваться его членством в
воркспейсе (`resolveStaffWorkspaceRole` уже есть), а не поиском клиентской
записи по почте.

**Приоритет:** средний. Обе ветки просыпаются от обычной работы владельца —
переименования ветки по умолчанию и первой роли в конфиге.

---

## TD-114: `roles` строкой в конфиге портала роняет блоб-роут в 500

**Где.** `backend/src/api/v2/modules/portal/code-guard.js`
(`collectRepoExposure`), `backend/src/api/v2/modules/portal/config-utils.js`
(`validatePortalConfig`, `filterByRole` в `roles.js`),
`backend/src/api/v2/modules/workspaces/workspace-templates.js:1820` и `:2687`.

**Что не так.** Заслон читает `page.roles` и `mod.roles` как массивы. Строка
проходит проверку `?.length` (у `'VIP'` длина 3) и дальше ведёт себя двояко —
замер 22.08.2026 прямым вызовом:

| конфиг | что выходит |
| --- | --- |
| `page.roles: 'VIP'` и `mod.roles: ['VIP']` | `TypeError: pageRoles.filter is not a function` → маршрут отвечает 500 |
| `page.roles: 'VIP'`, у модуля ролей нет | роли вырождаются в буквы: `roles = ["V","I","P"]`, `anonymous = false` |
| та же строка на экране (`filterByRole`) | модуль ВИДЕН: `'VIP'.includes('VIP')` истинно |

То есть экран строку принимает, а заслон на ней падает либо решает по буквам.

**Достижимо ли.** Через `POST /:db/portal/api/config` — нет:
`validatePortalConfig` (`config-utils.js:119` и `:143`) требует массив. Но
наложение шаблона пишет конфиг в `_v2_portal_config` **прямым INSERT без
проверки** — два места в `workspace-templates.js`, и содержимое приходит из
манифеста, то есть извне.

**Чем платим.** Раздел, который для владельца выглядит настроенным, отдаёт 500
на каждый файл компонента; в вырожденном случае — молча меняет круг допущенных.

**Что сделать.** Привести роли к массиву в одном месте на чтении (заслон и
`filterByRole` обязаны читать одинаково) либо прогонять `validatePortalConfig`
на пути наложения шаблона. Первое дешевле и закрывает уже записанные конфиги.

**Приоритет:** средний — 500 на живом разделе, причём с пути, который проверку
не проходит вовсе.
