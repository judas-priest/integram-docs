# Module: presence

**Path:** `src/api/v2/modules/presence/`
**Files:** `router.js`, `service.js`
**Base URL:** `/api/v2/:db/presence`
**Auth:** JWT required. POST: editor+ (плюс CSRF), GET: viewer+ — и для ленты по точке
роли мало: `GET /:db/presence` с парой `point_type`+`point_id` проходит ещё через
`canReadPoint` (грант `READ` на тип, а для записи — плюс построчное правило) и отвечает
403 `No read access to this point`. Лента по `actor` мимо этого судьи идёт — она собирает
события по многим точкам сразу, и в коде это названо оставшимся пробелом, а не задумкой.
**ADR:** [ADR-026](../../../docs/adr/026-presence-and-last-seen.md)

## Purpose

Журнал «кто был на рабочей точке и что там делал». Точка — таблица или запись.
События приходят двумя путями: WS-канал `point` (`ws.js`) и `POST /:db/presence`;
пишутся пакетами в `_v2_presence_log` (таблица на воркспейс) и хранятся 90 дней.

## Чем модуль НЕ является

**Здесь нет «последнего визита участника».** С 20.08.2026 эта величина живёт в
колонке `_v2_memberships.last_seen_at` и пишется `workspaceRoleMiddleware`
(`modules/workspaces/last-seen.js`), а не журналом присутствия. Разделение и его
причины — [ADR-026](../../../docs/adr/026-presence-and-last-seen.md).

Практическое следствие: **`actor` в этом журнале — не ключ к `_v2_users`.**
WS пишет актора с приставкой: `user:<username>`. Сравнивать `actor` с `username`
напрямую нельзя — именно на этом прежняя «последняя активность» в карточке
участника не показывалась никогда, даже когда события в журнале были бы.

Присутствие «прямо сейчас» тоже не хранится: оно живёт в открытых сокетах
(`ws._subscriptions`), журнал нужен для ленты точки, а не для ответа «кто здесь
сию секунду».

## REST Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/:db/presence` | Записать событие присутствия (в очередь пакетной вставки) |
| GET | `/:db/presence` | Лента событий по точке или по актору |

Оба маршрута перед работой зовут `ensureSideTables(pool, db, 'presence')` —
таблица заводится лениво.

### POST `/:db/presence`

Тело (zod-схема в `router.js`):

| Field | Type | Required | Ограничение | Description |
|-------|------|----------|-------------|-------------|
| `point_type` | enum | да | `POINT_TYPES` | `table` либо `record` |
| `point_id` | int | да | > 0 | ID точки |
| `kind` | enum | да | `PRESENCE_KINDS` | `intent`, `act`, `join`, `leave`, `help` |
| `zone` | string | нет | `COLUMN_WIDTHS.zone` | Зона интерфейса внутри точки |
| `what` | string | нет | ≤ 2000 | Что делает |
| `ref` | string | нет | `COLUMN_WIDTHS.ref` | Ссылочный контекст |
| `peer` | string | нет | `COLUMN_WIDTHS.peer` | Собеседник |

**`actor` в теле не принимается.** Сервер ставит его сам — `user:<username>`
вошедшего, ровно как WS-канал `point`. Тело с полем `actor` получает
`400 VALIDATION_ERROR`, а не молчаливое отбрасывание: присланное поле — не
опечатка, а попытка подписаться чужим именем, и участник с ролью editor подделывал
так чужое присутствие в ленте точки и в ленте актора.

Ширины строковых полей взяты из `COLUMN_WIDTHS` в `service.js` — того же
объявления, которым строится DDL. Числами они здесь не выписаны: второй перечень
разошёлся бы с колонкой молча. Исключение одно — `what`: её колонка `TEXT`,
своей ширины у неё нет, и 2000 тут ограничивают размер тела запроса, а не колонку.

Отвечает `201 { message: 'Event queued' }` — **сразу, до записи в базу.** Успех
ответа не означает, что строка легла: вставка отложенная, а её отказ уходит в лог.

### GET `/:db/presence`

Параметры запроса: `point_type` (из `POINT_TYPES`), `point_id`, `actor`
(≤ `COLUMN_WIDTHS.actor`), `limit` (1..200, по умолчанию 50), `offset`.

Нужно задать **либо** пару `point_type` + `point_id`, **либо** `actor`. Ни того,
ни другого — `400 VALIDATION_ERROR`.

Ответ: `{ data: [...строки журнала...], total: <число> }`.

## WebSocket Point Channel

Реализация — `ws.js`, клиент — `frontend/src/composables/usePointPresence.js`
(зовёт `views/data/ObjectView.vue`).

### Подписка — обычный кадр `subscribe`, а не собственный род кадра

```json
{ "type": "subscribe", "channel": "point", "pointType": "record", "pointId": 123 }
```

Ключ подписки: `point:<pointType>:<pointId>`.

`pointType` обязан быть из словаря `POINT_TYPES` в `presence/service.js` — сейчас
это ровно **`table`** и **`record`**. Другого рода точки нет; кадр с чем-то иным
получает в ответ `{"type":"error","code":"INVALID_PARAMS"}`. Тот же словарь читает
подписка на тему teamchat (`teamchat/subscriptions.js`) — второго списка заводить
нельзя.

`pointId` обязан быть целым положительным. Разбор ссылки на точку един
(`_parsePointRef`) и стоит на **трёх** входах: `subscribe`, `unsubscribe` и
кадр `point`.

**Подписка проходит через гейт доступа** `canSubscribePoint` (исполнение
[ADR-010](../../../docs/adr/010-ws-subscribe-authorization.md)): для `table` —
`checkGrant(id = typeId, t = typeId, 'READ')`, тот же судья, что у `listObjects`;
для `record` — `checkGrant` по типу записи плюс построчные правила
`checkRowPermission(..., 'READ', ...)`, как в `getObject`. Отказ:
`{"type":"error","code":"FORBIDDEN","message":"No read access to this point"}`.
Иначе участник, которому запись читать нельзя, видел бы в `presence-list` имена
смотрящих, светился там сам и получал свободный текст `what` / `ref` из чужих
кадров `intent`.

При успешной подписке сервер: рассылает подписчикам `presence-join`, пишет в
журнал событие `join`, отвечает подписавшемуся `{"type":"subscribed",...}` и
отдельным кадром шлёт ему `presence-list`.

### Отписку клиент шлёт сам

```json
{ "type": "unsubscribe", "channel": "point", "pointType": "record", "pointId": 123 }
```

`WsClient.unsubscribe()` во фронтенде серверу **не пишет** — он только чистит
локальную карту подписок. Пока клиент не отправит кадр `unsubscribe` явно, ни
`presence-leave` соседям, ни запись `leave` в журнал не появятся — до закрытия
сокета.

`leave` пишется в двух местах: на явный `unsubscribe` (если подписка
действительно была — иначе двойной уход) и в обработчике `close` для каждой
оставшейся подписки `point:*`.

### Операции канала

Клиентские операции идут кадром `{"type":"point","operation":...}`.

| Operation | Направление | Описание |
|-----------|-------------|----------|
| `presence-join` | server → client | Кто-то подписался на точку |
| `presence-leave` | server → client | Кто-то отписался или отвалился |
| `presence-list` | server → client | Кто сейчас на точке; шлётся подписавшемуся сразу после `subscribed` |
| `presence-ping` | client → server → other clients | Сердцебиение; сервер ретранслирует остальным подписчикам точки (кроме отправителя). В журнал **не** пишется |
| `intent` | client → server → other clients | Заявление о намерении. Ретранслируется остальным подписчикам с полями `what` и `ref`, и **пишется в журнал** родом `intent` |

Отправитель, не подписанный на точку, обслуживания не получает: `handlePointOp`
отвечает кадром ошибки `NOT_SUBSCRIBED` («Нет подписки на точку
`<pointType>:<pointId>` — сперва subscribe.») — так же, как каналы `objects`,
`documents`, `sandbox-collab`. Молчание неотличимо от доставки: клиент, потерявший
подписку при переподключении, слал намерения в пустоту и считал, что его видят.

### Форма кадра сервер → клиент

```json
{
  "type": "point",
  "operation": "presence-join",
  "pointType": "record",
  "pointId": 123,
  "userId": 42,
  "username": "alice",
  "ts": 1718300000000
}
```

`presence-list` вместо `userId`/`username` несёт массив `users:
[{ userId, username }]`. `intent` добавляет `what` и `ref`.

### Как событие попадает в журнал

Не «fire-and-forget REST POST», а прямой внутрипроцессный вызов:
`_logPointPresence` в `ws.js` подгружает `presence/service.js`, зовёт
`ensureSideTables(pool, db, 'presence')` и затем `logPresenceEvent(pool, db, {...})`
с `actor: 'user:<username>'`. Отказ уходит в лог (`[ws] presence log failed`) и
клиенту не виден.

Ленивое заведение таблицы здесь обязательно: события приходят по WS мимо REST-роута,
и в воркспейсе, где на `/:db/presence` ни разу не заходили, вставка падала бы на
«relation does not exist» (замер 20.08.2026 **расходится и требует перепроверки**:
комментарий в `ws.js` называет 11 схем примерно из ста, шапка
`scripts/backfill-presence-log.js` — 4 схемы из 206 воркспейсов; основания предпочесть
одно из чисел ни в коде, ни в истории нет).
`ensureSideTables` помнит обещание по тройке (пул, база, модуль) — звать на каждое
событие дёшево.

**Из WS рождаются только `join`, `leave`, `intent`.** Роды `act` и `help` доступны
лишь через REST — для фронтендов и автоматизаций (например, отметка о завершении
задачи или просьба о помощи).

## Пакетная запись, срок хранения, проверка входа

**Буфер.** `logPresenceEvent` кладёт событие в буфер, помеченный именем базы.
Сброс — по таймеру `FLUSH_INTERVAL_MS = 2000 мс` либо сразу при `FLUSH_MAX_SIZE = 50`
накопленных событиях. Сброс — **один многострочный INSERT на всю базу**. Буфер
помнит пул, с которым события пришли (важно при переезде воркспейса на другой DSN).
`flushPresence(db?)` сбрасывает принудительно — для выключения и тестов.

**Уборка.** `RETENTION_DAYS = 90`, `PRUNE_INTERVAL_MS = 60 * 60 * 1000` — не чаще
раза в час **на базу**. Уборка идёт хвостом сброса; её отказ сброс не роняет.
Срок уходит в запрос параметром через `make_interval(days => $1)` — `INTERVAL $1`
PostgreSQL не принимает.

**Проверка события — до буфера, а не на входах.** `_sanitizeEvent` внутри
`logPresenceEvent`:

- `point_id` обязан быть целым положительным в пределах `MAX_SAFE_INTEGER` — иначе
  событие отброшено;
- `kind` обязан быть из `PRESENCE_KINDS` — иначе отброшено (обрезать нельзя:
  негодное значение стало бы другим негодным);
- строковые поля из `TRUNCATED_COLUMNS` (`point_type`, `actor`, `zone`, `ref`,
  `peer`) **обрезаются** по ширине своей колонки с записью в лог — потерять поле
  лучше, чем потерять событие;
- пустые `point_type` или `actor` (они `NOT NULL`) — событие отброшено.

**Довод, ради которого проверка стоит именно здесь.** Вставка пакетная, и события
сняты из буфера через `splice(0)` ДО запроса. Значит одна негодная строка уносит
**всё** накопленное окно сброса для воркспейса, а не только себя. Замерено на
настоящей таблице: три честных события и одно с `NaN` в `point_id` в одном INSERT →
«неверный синтаксис для типа bigint: "NaN"», уцелело из трёх честных **ноль**. То же
дают NULL в NOT NULL, значение вне `CHECK` по `kind` и строка длиннее `VARCHAR`.
Писателей у журнала двое, поэтому проверка стоит у журнала, а не у каждого входа.

## DB Table

`_v2_presence_log` — таблица на воркспейс, заводится лениво
(`ensurePresenceTable`, через `ensureSideTables`).

**Ленивое заведение оставляет схемы без таблицы, пока в них не придут.** Замер
20.08.2026 **расходится и требует перепроверки**: шапка
`scripts/backfill-presence-log.js` называет 4 схемы из 206 воркспейсов, комментарий
в `ws.js` — 11 схем примерно из ста; расходятся и делимое, и делитель, а основания
предпочесть одно из чисел ни в коде, ни в истории нет. Досыпка по всем схемам —
`scripts/backfill-presence-log.js`: холостой прогон по умолчанию, настоящая работа
по `--apply`, DDL берётся из `ensurePresenceTable` (второго объявления нет),
идемпотентен, отказ на одном воркспейсе не рвёт обход, но попадает в итог и в код
возврата. Проверка обхода — `presence/__tests__/backfill-presence-log.test.js`
(лежит в модуле, а не рядом со скриптом: прогон vitest собирается по `src/**`).

Ширины строковых колонок объявлены **один раз** в `COLUMN_WIDTHS`
(`presence/service.js`), и DDL строится из них. По ним же обрезается вход.
Второго списка ширин заводить нельзя: разойдясь с DDL, он стерёг бы не ту ширину,
и это не всплыло бы до первой длинной строки на проде.

| Column | Type | Null | Description |
|--------|------|------|-------------|
| `id` | `BIGINT GENERATED ALWAYS AS IDENTITY` | PK | |
| `point_type` | `VARCHAR(64)` | NOT NULL | Род точки; словарь `POINT_TYPES` стерегут оба входа |
| `point_id` | `BIGINT` | NOT NULL | ID точки |
| `actor` | `VARCHAR(128)` | NOT NULL | `user:<username>`; ставит сервер на обоих входах |
| `kind` | `VARCHAR(16)` | NOT NULL | `CHECK (kind IN (...))`, список строится из `PRESENCE_KINDS` |
| `zone` | `VARCHAR(64)` | null | Зона интерфейса |
| `what` | `TEXT` | null | Что делает — единственная строковая колонка без ширины, поэтому её нет в `COLUMN_WIDTHS` |
| `ref` | `VARCHAR(256)` | null | Ссылочный контекст |
| `peer` | `VARCHAR(128)` | null | Собеседник |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, `DEFAULT NOW()` | |

Индексы — ровно под два запроса модуля:

| Index | Колонки | Кто читает |
|-------|---------|------------|
| `idx_presence_point` | `(point_type, point_id, created_at)` | `getPointFeed` |
| `idx_presence_actor` | `(actor, created_at)` | `getActorFeed` |

## Экспорты service.js

| Export | Что это |
|--------|---------|
| `PRESENCE_KINDS` | Единственный словарь родов события; из него строится `CHECK` в DDL и `z.enum` в роутере |
| `POINT_TYPES` | Единственный словарь родов точки (`table`, `record`); читают `ws.js` и `teamchat/subscriptions.js` |
| `COLUMN_WIDTHS` | Ширины строковых колонок; из них строится DDL и по ним обрезается вход |
| `RETENTION_DAYS`, `PRUNE_INTERVAL_MS` | Срок хранения и период уборки |
| `ensurePresenceTable(pool, db)` | DDL |
| `logPresenceEvent(pool, db, event)` | Проверить и поставить в очередь (синхронная, ничего не ждёт) |
| `flushPresence(db?)` | Принудительный сброс буфера |
| `getPointFeed(pool, db, {...})` | Лента точки |
| `getActorFeed(pool, db, {...})` | Лента актора |
