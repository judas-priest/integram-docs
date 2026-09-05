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

## Rate Limits (per execution)

| Category | Max calls | Bridge functions |
|----------|-----------|------------------|
| query | 10 | `query`, `getRecord`, `kag_search`, `search_decisions` |
| mutation | 10 | `createRecord`, `updateRecord`, `deleteRecord` |
| fetch | 5 | `fetch`, `browse` |
| ai | 3 | `ai` |
| teamchat | 5 | `sendTeamchatMessage` |
| agent | 0 (disabled) | `delegateToAgent` |

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

Function source code is cached for 1 minute (TTL). Cache is invalidated per-repo on git push events (`codespace.push`).

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

## Files

| File | Purpose |
|------|---------|
| `backend/src/api/v2/modules/codespace/server-functions.js` | Loader, cache, capability parser |
| `backend/src/api/v2/modules/codespace/server-fn-executor.js` | V8 sandbox executor |
| `backend/src/api/v2/modules/portal/router.js` | HTTP endpoint (`POST /api/fn/:repo/:name`) |
| `portal/composables/useCustomCodeApi.js` | `callFunction()` client method |
