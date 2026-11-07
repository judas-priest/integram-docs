# Tool Response Format

Формат применяется к тому, что `tool-executor.js` возвращает агенту через `JSON.stringify(...)`.
Хелперы в `tools/*.js` — внутренний слой, их return value агент не видит напрямую.

## Шаблоны

```js
// Create → всегда id + type + message
{ id: 123, type: "table", name: "Clients", message: "Таблица «Clients» создана." }

// Update / trigger → всегда id + type + message
{ id: 5, type: "table", message: "Таблица #5 обновлена." }

// List → всегда items + total
{ items: [...], total: 5 }

// Delete → только message
{ message: "Таблица «X» удалена." }

// Bulk create → ids array + counts
{ ids: [10, 11, 12], created: 3, errors: 0, message: "Создано 3 записи." }

// Error → error code + message
{ error: "NOT_FOUND", message: "Таблица #99 не найдена." }

// HITL pending (деструктивные операции)
{ status: "pending_confirmation", message: "..." }

// Job-flow (normalizer и подобные — не CRUD)
{ jobId: "abc", status: "queued", ... }
```

## Правила

- `id` — числовой PK из БД. Если нет числового PK (normalizer job и т.п.) — используй именованное поле (`jobId`).
- `type` — строчный тип сущности: `table`, `column`, `object`, `report`, `document`, `block`, `folder`, `tag`, `automation`, `webhook`, `form`, `dashboard`, `view`, `role`, `row_rule`, `connector`, `computed`, `comment`, `backup`, `job`, `grant`, `member`.
- `message` — обязателен при мутациях. Human-readable, используется агентом для reasoning.
- Нет `success: true` и `ok: true` — эти поля не несут информации для агента.
- Нет legacy-ключей в списках (`tables`, `reports`) — всегда `items`.

## Workspace tools

Workspace tools (`_v2_workspace_tools`) return whatever the user code returns — there is no enforced schema. The sandbox serializes the return value via `JSON.stringify()`. If the tool code returns `undefined` or throws, the executor returns an error string.

```js
// Workspace tool response — no enforced format
// The code's return value is serialized directly:
return { range_km: 55.6 };
// → agent receives: '{"range_km":55.6}'
```

Platform tool response format rules (id, type, message, items) do NOT apply to workspace tools.

## Как проверять

Нарушение — только если `tool-executor.js` делает `return JSON.stringify(result)` напрямую,
а `result` содержит `ok`/`success`. Если tool-executor оборачивает результат сам или уходит
в HITL — хелпер может возвращать что угодно, агент это не видит.
