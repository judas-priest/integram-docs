# ADR-005: Event bus для межмодульного взаимодействия

**Статус:** accepted

## Контекст

Когда объект создаётся, нужно: записать в audit log, обновить vector embeddings, доставить webhook, запустить автоматизации, разослать WebSocket-событие, обновить граф. Прямой вызов всех этих сервисов из `objects/service.js` создаст жёсткую связь между модулями.

## Решение

Singleton `EventEmitter` — `utils/event-bus.js`. Producers эмитируют события, consumers подписываются независимо.

```js
// producer (objects/service.js)
bus.emit('object.created', { db, pool, objectId, typeId, ... });

// consumers (audit-listeners.js, embedding-listeners.js, ws-listeners.js, ...)
bus.on('object.created', async (data) => { ... });
```

Каждое событие автоматически получает `_event`, `_ts`, `_cid` (correlation ID).

Для fire-and-forget операций, которые не должны валить caller при ошибке:
```js
sideEffect('label', asyncOperation(args));
```

`sideEffect` никогда не бросает, только логирует ошибку.

**Что идёт через bus:** реакции на доменные события — audit, embeddings, WebSocket broadcast, graph edges, webhooks, автоматизации.

**Что не идёт через bus:** прямые вызовы сервисов для получения результата — automations импортирует notifications и connectors напрямую, tool-executor вызывает сервисы напрямую. Это нормально.

## Последствия

- Добавить нового подписчика = новый listener-файл, producer не трогать
- Максимум 50 listeners на bus (защита от утечек: `bus.setMaxListeners(50)`)
- Подписчики инициализируются при старте сервера в `src/api/v2/index.js`
- Automations при выполнении action пишет SQL в EAV напрямую (не через objects/service), чтобы не вызывать рекурсивные события
