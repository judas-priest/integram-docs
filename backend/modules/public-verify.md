# Module: public-verify

**Path:** `src/api/v2/modules/public-verify/`
**Files:** `router.js`, `service.js`, `checkpoint.js`, `checkpoint-store.js`, `checkpoint-worker.js`
**Base URL:** `/api/v2/public/verify/...` (анонимный), `/api/v2/:db/public-verify/...` (admin)
**Auth:** публичные GET — анонимно (rate limit по IP); конфиг воркспейса — JWT + роль `admin`.

## Purpose

Публичная проверка «решений» воркспейса (задача + протокол + вердикт) по пересчитанной хеш-цепочке: третья сторона без доступа к БД убеждается, что вердикт не переписан. Состояние подписывается ed25519-ключом, живущим в env сервера (вне БД), подписанные checkpoint'ы копятся в глобальной системной таблице и раздаются публичным feed-эндпоинтом. Перезапись схемы воркспейса не даёт «зелёной» проверки и не трогает ни ключ, ни накопленный лог.

## Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/public/verify/:db/:subjectId` | аноним | Полная проверка решения: пересчёт цепочки вердиктов + при активном ключе — подписанный checkpoint в ответе |
| GET | `/public/checkpoint/:db[?limit=N]` | аноним | Публичный feed: без `limit` (или `limit=1`) — последний checkpoint (объект); `limit>1` — история (массив, новее сверху; до 200) |
| GET | `/:db/public-verify/config` | admin | Текущий конфиг секций |
| PUT | `/:db/public-verify/config` | admin | Сохранить конфиг целиком; частичный — отвергается (422) |

## Публичный feed и webhook (checkpoint)

### Environment

Фича включается ключом подписи; без переменных checkpoint'а нет, verify работает как раньше.

| Переменная | Смысл | Не задана → |
| --- | --- | --- |
| `PUBLIC_VERIFY_SIGN_KEY` | PEM приватного ключа ed25519 (pkcs8) | checkpoint'а нет (фича выключена), verify как сейчас |
| `PUBLIC_VERIFY_SIGN_KID` | идентификатор ключа (напр. `2026-10`) | вместе с KEY → воркер НЕ заводится (лог.error, процесс жив) |
| `PUBLIC_VERIFY_CHECKPOINT_INTERVAL_MIN` | период cron-воркера в минутах (по умолчанию 15, зажимается в 1..59) | 15 |

Воркер очереди `public-verify-checkpoint` заводится в `backend/scripts/start-worker.js` ТОЛЬКО при полной паре KEY+KID; расписание — своя очередь BullMQ (повторяемый job `*/N * * * *`), не общий контур автоматизаций.

### Формат checkpoint

```json
{ "head": "<64 hex — голова пересчитанной хеш-цепочки>",
  "links": 12, "anchors": "<64 hex — дайджест якорей>",
  "ts": 1791970000000, "kid": "2026-10", "signature": "<base64 ed25519>" }
```

Подпись ставится на каноническую строку `db|head|links|anchors|ts|kid` (db берётся из адреса feed-запроса). Разрыв цепочки (`chain.reason` в verify-ответе) сознательно НЕ входит в подпись — checkpoint закрепляет фактическое состояние; сам разрыв виден получателю в verify-ответе.

Cron-воркер каждые N минут подписывает состояние каждой настроенной БД и дописывает checkpoint в глобальную таблицу `_v2_public_verify_checkpoints` append-only: повторная запись делается только при ИЗМЕНЁННОМ `head` — история лога не заполняется дублями, webhook-получатели не спамятся.

### Как проверить третьей стороне

1. Скачать checkpoint из feed: `GET /api/v2/public/checkpoint/:db`.
2. Взять открытый ключ ed25519 — владелец публикует его ВНЕ БД (тот же `kid`).
3. Проверить подпись: `crypto.verify(null, Buffer.from(`${db}|${head}|${links}|${anchors}|${ts}|${kid}`), publicKey, Buffer.from(signature, 'base64'))` — должна быть `true`.
4. Сверить `head`/`anchors` с живым verify-ответом `GET /api/v2/public/verify/:db/:subjectId` — совпадение означает, что текущее состояние совпадает с подписанным.

Получатель хранит копию сам (канон C2SP tlog-tiles: лог раздаёт checkpoint по HTTP, наблюдатели тянут); feed — источник истины.

### Webhook-контракт

Админ воркспейса перечисляет адреса в конфиге: `PUT /:db/public-verify/config` с секцией `checkpoints.webhooks` — массив https-URL (до 5, без дубликатов; `[]` снимает подписку; не-массив/не-https — 422). При ИЗМЕНЕНИИ состояния воркер делает на каждый адрес:

```
POST <webhook>
Content-Type: application/json
User-Agent: integram-public-verify

{ "db": "<db>", "checkpoint": { ...checkpoint... } }
```

Push — fire-and-forget без retry: недоступный получатель получит следующий checkpoint при следующем изменении состояния, недоставленные изменения всегда видны в feed. Дубликатов нет — push только на новый `head`.

### Ротация ключа

Новый ключ = новый `kid` в env (`PUBLIC_VERIFY_SIGN_KEY` + `PUBLIC_VERIFY_SIGN_KID` парой); старые checkpoint'ы остаются валидными под своим `kid` — открытый ключ прежнего `kid` публикуется наряду с новым. Пара KEY без KID (и наоборот) воркер не заводит — процесс не падает.

### Ограничение модели

Защита покрывает перезапись схемы/данных воркспейса: ключ и лог checkpoint'ов живут вне области. Компрометация сервера целиком — вне модели: следующий уровень — внешний держатель копий feed'а; RFC 3161 (доверенная метка времени) — при необходимости.
