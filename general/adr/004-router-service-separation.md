# ADR-004: router.js — HTTP only, service.js — бизнес-логика

**Статус:** accepted

## Контекст

Express допускает писать логику прямо в роутере. В больших модулях (objects ~3300 строк, portal ~4500 строк) это делает код нетестируемым и переплетает HTTP-детали с бизнес-логикой.

## Решение

Каждый модуль (`backend/src/api/v2/modules/<name>/`) состоит из двух файлов:

**`router.js`** — HTTP-слой только:
- Парсинг `req.params`, `req.body`, `req.query`
- Вызов сервиса
- `res.json(result)` или `next(err)`
- Никакого SQL, никакой логики

**`service.js`** — бизнес-логика только:
- Параметры — обычные JS-значения, не `req`/`res`
- Возвращает данные или бросает `AppError`
- Не знает о HTTP

Ошибки: только `throw new AppError(CODE, message, httpStatus)`. Никогда `new Error()`.

## Последствия

- `service.js` тестируется без HTTP-контекста (unit-тесты с `vi.mock`)
- Один сервис может вызываться из нескольких мест: router, tool-executor, другие сервисы
- `req` и `res` никогда не импортируются в `service.js`
- Новый модуль = обязательно оба файла, подключить в `src/api/v2/index.js`
