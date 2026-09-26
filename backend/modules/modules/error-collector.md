# error-collector

Сбор ошибок с фронтенда и бэкенда + приём баг-репортов от пользователей MCP
(`report_platform_issue`). Дедуп по отпечатку: одна проблема — одна строка
журнала со счётчиком вхождений.

## Таблица

`_v2_error_reports` — глобальная (не воркспейсная), создаётся кодом
`ensureErrorReportsTable(pool)` (CREATE IF NOT EXISTS + ALTER ADD COLUMN IF NOT
EXISTS; системы миграций в модуле нет — ensure идемпотентен и вызывается перед
записью). Колонки: `fingerprint VARCHAR(64) UNIQUE`, `source`
(`frontend`/`backend`/`mcp`), `error`, `stack`, `file`, `line`, `page_url`,
`username`, `workspace`, `browser`, `count`, `first_seen`, `last_seen`,
`pm_issue_id`, `pm_workspace`.

Fingerprint = sha256(`message|file|line`)[:32] — детерминирован: одинаковый
отказ даёт одинаковый ключ, `ON CONFLICT` увеличивает `count` и обновляет
`last_seen`.

## Маршруты

Смонтирован глобально, вне `/:db`, до `requireJwt`
(`api/v2/index.js`): `router.use('/error-reports', …)`.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/error-reports` | jwtAuth optional + rateLimit 20/мин/IP | Пакетный приём ошибок фронта (`events[]`, до 50). Шум (`Script error`, extension-скрипты) отфильтровывается. |
| POST | `/error-reports/mcp` | jwtAuth (обязателен) + rateLimit 10/час/юзер | Баг-репорт пользователя MCP (см. ниже). |

## Пайплайн MCP-репортов (`POST /error-reports/mcp`)

Тул `report_platform_issue` (TOOLS: group `core`, TIER_HIGH — каждое отправление
через HITL-подтверждение) вызывает маршрут с `toolName`, `title`,
`whatHappened`, `errorCode?`, `errorMessage?`, `category?`
(bug/docs/missing_capability/ux), `severity?`, `workspaceSlug`, `mcpVersion?`.

1. `redactSecrets` (`shared/redact.js`) вырезает секреты из пользовательских
   полей (bearer/`smk_`/JWT/пароли/email/длинный hex-b64), обрезает 2000 знаков.
   Email репортёра не редактируется — это серверная личность (JWT), она же ключ
   дневного лимита.
2. Fingerprint по `toolName + errorCode` (без кода — первые 120 знаков
   описания). Lookup в `_v2_error_reports` по `source='mcp'`.
3. **Первое появление**: PM-задача в intake-воркспейсе (`FEEDBACK_DB`, по
   умолчанию ai2o) от сервисного бота (`FEEDBACK_BOT_USER_ID`; репортер — бот,
   юзер прописан в описании) через `pm/service.createIssue` — образец
   межворкспейсной записи в `orgs/pm-aggregation.js`. Метки: `feedback`,
   `sig:<hash12>`, категория. Затем INSERT строки дедупа с `pm_issue_id`.
4. **Повтор**: `count+1`, в описании задачи обновляется «Вхождений: N», закрытая
   задача переоткрывается (`status: 'todo'`). Сбой PM-апдейта не теряет отчёт —
   счётчик растёт, в лог warn.
5. **Лимиты**: новых PM-задач ≤ 3/сутки на репортёра (сверх — вхождение
   учитывается в журнале, задача не создаётся); правовые отказы
   (FORBIDDEN/NO_READ_ACCESS) багом не считаются — их за это не отправляют
   (правило в инструкциях mcp-server).

Intake-воркспейс обязан существовать (строка в `_v2_workspaces`), иначе маршрут
громко отказывает; PM-таблицы создаются лениво через `ensureSideTables`.

## Отчёты в PM

Задачи живут в intake-воркспейсе и видны пользователю только через номер issue,
который тул возвращает. Полный журнал вхождений читается штатными маршрутами
этого модуля. Триггеры вызова и анти-триггеры — в инструкциях mcp-server
(раздел «Reporting platform issues»).
