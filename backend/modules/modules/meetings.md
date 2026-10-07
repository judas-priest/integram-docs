# Module: meetings

**Path:** `src/api/v2/modules/meetings/`
**Files:** `router.js`, `service.js`, `templates.js`
**Base URL:** `/api/v2/:db/meetings/...`
**Auth:** JWT; все маршруты — admin + CSRF на запись (setup/teardown/PUT config), чтение (GET config/status) — admin без CSRF.

## Назначение

Конвейер «TG-запись → расшифровка → протокол → одобрение → PM-задача + голосовое», переиспользуемый между воркспейсами. Админ один раз вызывает `setup` с конфигом: модуль резолвит алиасы колонок в reqId и устанавливает в воркспейс свои автоматизации с префиксом `meetings: ` (intake — приём TG-сообщений с файлом; protocol — реакция на смену статуса встречи). Конфиг хранится per-workspace в `_v2_meetings_config` (одна строка, JSONB).

## Что ядерное, что модуль

Модуль владеет только **оркестрацией** — своими автоматизациями `meetings: *`. Ядерные механизмы им не дублируются:

- **расшифровка** — files/doc-processor: колонка «Транскрипт» в таблице сообщений заполняется штатным пайплайном, запись строки диспатчит штатное `on_update`;
- **дедуп** — portal intake (повторная подача того же сообщения не создаёт вторую встречу).

## Схема конфига

| Поле | Описание |
|------|----------|
| `botId` | ID Telegram-бота (таблица ботов), который принимает голосовые записи. |
| `chatId` | Чат, из которого принимаются сообщения (числовой Telegram chat id). |
| `tg.tableId` | typeId таблицы входящих сообщений. |
| `tg.cols` | 12 алиасов колонок сообщения: `date, chat, chatId, author, userId, text, type, messageId, fileName, fileId, fileMeta, transcript`. |
| `meet.tableId` | typeId таблицы «Встречи». |
| `meet.statusValue` | Значение колонки статуса, включающее протоколирование (триггер `on_update`). |
| `meet.cols` | 8 алиасов: `status, record, recognized, merged, protocol, verdict, summary, link`. |
| `participants` | `null` либо `{tableId, cols, createIfMissing}` — таблица участников: resolv алиасов `cols`, `createIfMissing` — создавать отсутствующих участников. |
| `approveUsername` | Имя пользователя, чьё одобрение требуется перед созданием PM-задачи. |
| `issueAssigneeId` | ID исполнителя, проставляемый в создаваемой PM-задаче (опционально). |
| `issueLabels` | Метки через запятую для создаваемой PM-задачи. |
| `portalBase` | База URL портала — из него пекутся ссылки на документы в полях действий. |
| `dict` | Промпт-словарь терминов для расшифровки/протокола (свободный текст, опционально). |

Алиасы в конфиге — имена колонок байт-в-байт; `resolveReqIds` отвергает setup, если алиас не найден или подходит под несколько колонок.

## Порядок включения

1. Создать таблицу «Встречи» с каноническими алиасами (см. `meet.cols`) и таблицу сообщений с колонкой «Транскрипт» (см. `tg.cols`).
2. `POST /:db/meetings/setup` (admin, csrf) с конфигом — модуль резолвит reqId, устанавливает автоматизации `meetings: intake (bot N)` и `meetings: protocol`, сохраняет конфиг.
3. `GET /:db/meetings/status` (admin) — `{configured, config, automations: [{id, name, active}]}`.

Повторный `setup` обновляет автоматизации по имени (upsert: SELECT-then-write — `UNIQUE` на `name` в `_v2_automations` нет).

**ВАЖНО:** doc-processor ищет колонку транскрипта по `LIKE` — держать алиасы уникальными внутри таблицы (без колонок «Транскрипт 2»: на коллизии подстрок штатный поиск молча берёт первую строку). `resolveReqIds` на setup такие случаи отвергает для колонок конфига.

## Teardown

`POST /:db/meetings/teardown` (admin, csrf) — деактивация всех автоматизаций по префиксу `meetings: %`; на каждую деактивированную строку эмитится `automation.saved` с `active: false`, что снимает BullMQ-ключ расписания (ядро, `worker.js onAutomationSaved`). Возвращает `{deactivated: n}`. Конфиг и записи не трогаются.

## Идемпотентность и границы

- `UNIQUE` на `name` в `_v2_automations` нет, поэтому upsert — SELECT-then-write; гонка двух одновременных setup теоретически возможна, операция разовая — защита не заводилась.
- Повторный setup с **другим** `botId` создаст вторую intake-автоматизацию (`meetings: intake (bot N)` — имя зависит от бота); старая останется активной — деактивировать вручную или через teardown + setup.
- ReplyTo-колонка в канонический конфиг не входит (осознанное упрощение).
