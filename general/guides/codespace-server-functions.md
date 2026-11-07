# Codespace Server Functions

Server-side JavaScript functions stored in codespace repos, executed in isolated V8 sandboxes. Called from portal custom code via `api.callFunction()`.

## Convention

Place `.js` files in the `api/` directory of a codespace repo:

```
portal-components/
  api/
    toggle-collected.js
    calculate-price.js
  main.vue
  ...
```

Function name = filename without `.js`. Names must match `/^[a-z][a-z0-9-]{0,62}$/`.

## Capabilities Header

Declare capabilities in the first comment line. Default if omitted: `query`.

```js
// capabilities: query, write, fetch
const items = await query(args.typeId, { limit: 100 });
// ...
```

## Bridge Functions

### query(typeId, opts) -> Array

Query EAV records. Requisites are flattened: `item[String(reqId)]`.

```js
const items = await query(42, { limit: 10, search: 'test' });
// items[0]["123"] === "field value"
```

### getRecord(objectId) -> Object

Get a single record. Requisites are nested: `item.requisites[String(reqId)]`.

```js
const rec = await getRecord(1001);
// rec.requisites["123"] === "field value"
```

### createRecord(typeId, { name, fields, parentId }) -> Number

Create a record. Returns the new object ID. `fields` is `{ [reqId]: value }`.

```js
const newId = await createRecord(42, {
  name: 'New item',
  fields: { '123': 'value', '124': '500' },
});
```

#### `up` / `parentId`: родитель-запись и невидимый `up=0`

`createRecord` пишет `up = parentId`, когда он передан, и `up = 1` (корень
области), когда опущен; переданный `parentId: 0` тоже ляжет как `1` (falsy).
Записи справочников («справочник» = дочерние строки) живут с
`up=<id родителя-записи>`: создайте родителя, потом
`createRecord(tid, { name, parentId: ownerId })`.

Строка с `up = 0` — это ОПРЕДЕЛЕНИЕ СХЕМЫ (тип/колонка), а не данные: все
чтения (`listObjects`, песочничный `query`, MCP) строят WHERE через
`objectWhere()` (`backend/src/shared/sql-guards.js`), который требует
`up != 0`, — такая строка невидима ВО ВСЕХ запросах. 14.10.2026 на проде
записи справочника, залитые с `up=0`, не появились в таблице. Данным —
никогда не писать `up=0`.

### updateRecord(objectId, { fields }) -> null

Update record fields. `fields` is `{ [reqId]: value }`.

```js
await updateRecord(1001, { fields: { '123': 'new value' } });
```

### deleteRecord(objectId) -> null

Delete a record.

```js
await deleteRecord(1001);
```

### fetch(url, opts) -> { status, body, ok }

HTTP fetch. Requires `fetch` capability.

```js
// capabilities: fetch
const res = await fetch('https://api.example.com/data', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ key: 'value' }),
});
if (res.ok) return JSON.parse(res.body);
```

### ai(prompt, model?, opts?) → string

Один вызов LLM. Возвращает СТРОКУ — `content` модели; объект ответа наружу
не отдаётся никогда. Бюджет: 3 вызова (`ai`) на прогон СФ; в automations-профиле — 10.

- `prompt` — строка (приводится `String()`), уходит единственным user-сообщением; `temperature: 0.7`.
- `model` — алиас маршрутизатора, `'fast'` по умолчанию.
- `opts` — необязательный `{ maxTokens, timeoutMs }`:
  - `maxTokens`: умолчание 2000 (env `AUTOMATION_AI_MAX_TOKENS`), жим `[1, 8000]`;
  - `timeoutMs`: умолчание 30 000 (env `AUTOMATION_AI_TIMEOUT_MS`), жим `[1000, 120000]`.

Поведение при обрыве:

- пустой `content` при `finish_reason === 'length'` → ОДИН ретрай с удвоенным
  `maxTokens` (reasoning-модель тратит бюджет на thinking и не договаривает;
  OmniRoute #3587). Ретрая нет, если `maxTokens * 2 > 8000`.
- `reasoning_content` возвращается ТОЛЬКО при `finish_reason === 'stop'`
  (последние 4000 знаков): часть провайдеров кладёт ответ в мышление и
  завершается нормально. При `'length'`/`'max_tokens'` и прочих мышление
  ответом НЕ считается (PM-1108: протокол встреч принял размышления GLM-5.3
  за «решений не зафиксировано»).
- остальные пустые ответы → `''` (не исключение) + предупреждение в лог;
  скриптовые фолбэки живут.
- HTTP-отказ или `data.error` провайдера — исключение `AI error: HTTP N`
  наружу скрипта.

```js
// capabilities: ai
const verdict = await ai('Ответь строго JSON {"ok":true|false}. Текст: ' + args.text, 'fast', { maxTokens: 4000 });
if (verdict === '') return { skipped: 'пустой ответ модели' };
```

Ловушка времени: потолок одного вызова — 120 с, а СФ с мостами по умолчанию
живёт 15 с. Долгий вызов сопровождай шапкой `// timeout: 150000`, иначе
оборвётся функция, а не запрос к модели.

## Rate Limits (per execution)

| Category | Max calls | Bridge functions |
|----------|-----------|------------------|
| query | 30 | `query`, `queryPage`, `getRecord`, `kag_search`, `search_decisions` |
| mutation | 10 | `createRecord`, `updateRecord`, `deleteRecord` |
| fetch | 5 | `fetch`, `browse` |
| ai | 3 | `ai` |
| teamchat | 5 | `sendTeamchatMessage` |
| pm | 30 | `createRecord`/`updateRecord` над PM-таблицами (P4-T0: фондовые ритуалы шлют по сообщению на шаг) |
| tg | 30 | исходящие Telegram-мосты |
| files | 5 | файловые мосты |
| agent | 0 (disabled) | `delegateToAgent` |

## PM-мосты песочницы (capability `pm`) {#pm-bridges} <!-- PM-1004 -->

Объявление в заголовке функции: `// capabilities: pm` — даёт пять мостов из
`backend/src/api/v2/modules/automations/sandbox-pm.js`. Бюджет: 30 вызовов
на прогон. Все мосты работают в контексте СВОЕГО воркспейса (db подставляется
рантаймом); pmQuery/pmCreate исторически описаны в ADR-034.

### pmQuery(opts) → { total, items: [...] }

Чтение задач. Обязателен хотя бы один фильтр — `assigneeId`, `reporterId`
или `q` (иначе `VALIDATION_ERROR pmQuery`). Без фильтра «все задачи области»
не отдаются.

### pmCreate(opts) → { id, number }

Создание задачи. Обязательны `title`, `assigneeId`, `reporterId` —
задач-сирот без исполнителя и постановщика мост не создаёт.

### pmInfo(opts) → { issue, parent, children, comments, commentsTotal }

Карточка задачи по НОМЕРУ: `{ number: 1071 }`. Обязателен `number`,
иначе `VALIDATION_ERROR pmInfo`. Несуществующий номер → `null` (не ошибка).
`commentsLimit` — сколько последних комментариев вернуть (умолчание 3,
потолок 50). `children` — подзадачи (до 50), `parent` — `{ number, title }`
родителя, если жив.

### pmUpdate(opts) → результат updateIssue

Машинный close-интент: `{ id, status }`, где `status` — ТОЛЬКО `done` или
`canceled` (иначе `VALIDATION_ERROR`). Права не судятся: вызывающий скрипт
сам сверил исполнителя/постановщика по чату; актор в истории — сентинел 0
(«не пользователь»). Описания/priority из песочницы не правятся.

### pmAddComment(opts) → результат addComment

`{ issueId, body }`, оба обязательны (пустое тело — `VALIDATION_ERROR`).
Автор — РЕАЛЬНЫЙ бот-член воркспейса: детерминированно «🤖 Машинка…», fallback —
старейший бот членства; если ботов нет — `VALIDATION_ERROR` («машинный
пользователь воркспейса не найден»). Комментарий с несуществующим автором
не был бы виден никому (INNER JOIN на _v2_users в listComments).

## Frontend Usage

From portal custom code components:

```js
const result = await props.api.callFunction('toggle-collected', { itemId: 42 });
// Default repo: 'portal-components'
// Custom repo:
const result = await props.api.callFunction('calculate-price', { items }, { repo: 'my-repo' });
```

Calls `POST /:db/portal/api/fn/:repo/:name`. Rate limit: 60/min per user per workspace.

## Cache

Function source code is cached for 1 minute (TTL). Кэш сбрасывается при ЛЮБОЙ записи в репозиторий: push, `commit_file`/REST-запись, merge PR — не только на `codespace.push`.

## Объявленный таймаут (`// timeout:`)

Первая строка-шапка `// timeout: N` задаёт таймаут функции в МИЛЛИСЕКУНДАХ (не секундах!), зажимается в [5000, 180000]. Без шапки — профильные 15 с (с мостами) / 5 с (чистая). `// timeout: 60` — это 60 мс → поднимется до минимума 5 с.

## Security

| Property | Value |
|----------|-------|
| Isolation | Dedicated V8 isolate (isolated-vm), fresh per request |
| Memory limit | 64 MB per isolate |
| Timeout (bridged) | 15 seconds |
| Timeout (pure, no capabilities) | 5 seconds |
| OOM handling | Graceful — isolate disposed, no `process.exit` |
| Auth | `portalAuth('admin', 'owner', 'editor')` — portal JWT required |

Each request gets a fresh isolate (never reused). On OOM, the isolate is disposed gracefully without crashing the host process — unlike workspace-tools which call `process.exit(1)`.

## Example

`api/toggle-collected.js`:

```js
// capabilities: query, write
const rec = await getRecord(args.itemId);
const current = rec.requisites[String(args.collectedReqId)] || '0';
const next = current === '1' ? '0' : '1';
await updateRecord(args.itemId, { fields: { [String(args.collectedReqId)]: next } });
return { toggled: next === '1' };
```

## Caller Identity

Every function body can read the `caller` global — a snapshot of who invoked it:

```js
// capabilities: query
return {
  you: caller.email || caller.username,
  elevated: caller.elevated, // true only for automation invocations
};
```

Fields: `id`, `username`, `email`, `role`, `roleId`, `source` (`'server-fn'` from the portal route, `'ai-tool'` from the AI tool; automations set no source), `elevated` (`true` only for automations, which run with `role: 'admin'`).

Authorization rules:

- Server-side role checks belong here, per invocation — never rely on the portal UI hiding a button.
- Do NOT build permissions on `caller.role` alone: the role string describes the platform role (owner/admin/editor/viewer), and automation callers arrive as elevated admins. Prove domain roles (e.g. "project manager") against your own workspace data — e.g. a team table matched by `caller.email`.

Example — a `whoami` function:

```js
// capabilities: query
return { caller: { id: caller.id, username: caller.username, email: caller.email, source: caller.source, elevated: caller.elevated } };
```

## Files

| File | Purpose |
|------|---------|
| `backend/src/api/v2/modules/codespace/server-functions.js` | Loader, cache, capability parser |
| `backend/src/api/v2/modules/codespace/server-fn-executor.js` | V8 sandbox executor |
| `backend/src/api/v2/modules/portal/router.js` | HTTP endpoint (`POST /api/fn/:repo/:name`) |
| `portal/composables/useCustomCodeApi.js` | `callFunction()` client method |
