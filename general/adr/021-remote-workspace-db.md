# ADR-021: Remote Workspace Databases

**Статус:** accepted

## Контекст

Все воркспейсы хранят данные в одном PostgreSQL-инстансе. Для enterprise-клиентов и тяжёлых воркспейсов нужна физическая изоляция — возможность вынести данные воркспейса на отдельный сервер.

## Решение

### Pool Manager

`backend/src/shared/pool-manager.js` кеширует `pg.Pool` инстансы по connection string (ключ: `host:port/database@user`). Два воркспейса на одном сервере — один пул.

- `getPool()` — всегда возвращает дефолтный (основной) пул. Используется для глобальных таблиц.
- `getPoolForDb(dbName)` — проверяет `dbToDsn` Map, возвращает remote pool или дефолтный.
- `registerRemoteDsn(dbName, dsn)` / `unregisterRemoteDsn(dbName)` — управление регистрацией.
- `testConnection(dsn)` — тестовое подключение перед сохранением.
- `evictIdle()` — eviction пулов с `totalCount === 0`, запускается по таймеру каждые 5 мин.

### Конфигурация

Колонка `_v2_workspaces.remote_dsn` (JSONB, DEFAULT NULL). Формат: `{host, port, database, user, password}`.

### Маршрутизация

- `dbMiddleware` → `getPoolForDb(resolvedDb)` → `req.pool` (workspace), `req.sysPool` (main).
- Workers → `getPoolForDb(job.data.db)`.
- WebSocket → `ws._pool = await getPoolForDb(resolvedDb)`.
- Listeners с `ev.pool` из event bus — уже корректный pool.
- Listeners с прямым `getPool()` для workspace данных → заменены на `getPoolForDb(ev.db)`.

### API

- `POST /workspaces/:slug/test-remote-dsn` — тест DSN.
- `PATCH /workspaces/:slug/remote-dsn` — сохранение (с обязательным тестом перед сохранением).

### Жизненный цикл

- При PATCH remote_dsn → bootstrap schema + EAV + satellite tables на remote сервере.
- При удалении воркспейса → DROP SCHEMA на правильном сервере, затем unregister.
- Клонирование remote воркспейса → заблокировано (cross-server copy не реализован).

## Последствия

- **Нельзя:** клонировать remote воркспейсы, делать cross-server JOIN в отчётах.
- **Упрощается:** физическая изоляция данных клиента, независимые бэкапы.
- **Ограничение:** worker-процесс (`start-worker.js`) не загружает remote DSN при старте — требует доработки при реальном использовании remote.
- **Пароль** хранится в plaintext JSONB. Планируется AES-шифрование в v2.
