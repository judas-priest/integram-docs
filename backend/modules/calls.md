# calls module — Конференции

1-1 звонки и комнатные конференции (до 12 участников), гостевые ссылки-приглашения (как Zoom/Телемост), live-транскрипция (Whisper), голосовые команды «Агент, …».

**Гейт модуля.** REST-роуты стоят за `requireModule('calls', { defaultEnabled: false })` — нужен явный `settings.modules.calls === true`, иначе 403 `MODULE_DISABLED` (не настроен = выключен, fail-closed; TD-111). WS-канал `calls` гасится только при ЯВНОМ `modules.calls === false` (`ws.js`).

## Files

| Файл | Роль |
|------|------|
| `router.js` | REST: `turn-credentials`, `signal-token`, `history`, `audio-chunk`, `voice-command` |
| `guest-links.js` | Ссылки-приглашения + `guest-token`; экспортирует `verifyGuestReq` / `guestCallsEnabled` |
| `signalling.js` | WS-обработчик канала `calls` (direct + room), гостевые запреты |
| `service.js` | История звонков/участники, переходы статусов |
| `call-timeout.js` | `CallTimeoutManager` — 30-секундный ring timeout |
| `turn-credentials.js` | Эфемерные HMAC-SHA1 TURN-креды (REST API, draft-uberti) |
| `migrations.js` | Идемпотентное создание satellite-таблиц |

## REST endpoints (под `/:db/calls`)

| Method | Path | Auth | Назначение |
|---|---|---|---|
| GET | `/turn-credentials` | member | Эфемерные TURN-креды. Без `TURN_SHARED_SECRET` → **200 с пустыми кредами** (`urls: []`, ttl 0), не ошибка: фронт работает через STUN/публичный IP |
| GET | `/signal-token` | member | Workspace-scoped JWT (1h) `{sub, email, db}` для Go signal server; сервер сверяет claim `db` |
| GET | `/history?limit=50` | member | Звонки, где пользователь инициатор или цель; `limit` капается на **200** |
| POST | `/audio-chunk` | member + гость | Чанк webm/opus (multer memoryStorage, ≤10 МБ, поле `audio`) → Whisper → `{text}`. Гость: guest JWT (`verifyGuestReq` — claim `db` обязан совпасть с базой маршрута) + гейт `guestCallsEnabled`, иначе 401/403 |
| POST | `/voice-command` | member only | Голосовая команда «Агент, …» — см. ниже. Гостю не открывается |
| GET \| POST | `/calls/links` | member | Список активных ссылок (до 50) / создание `{roomName?, expiresInHours?, maxUses?}` → `{token, url: /public/call/:db/:token}`; token — 8 симв. base64url |
| DELETE | `/calls/links/:token` | создатель или админ | Отзыв (`revoked = true`); создатель ссылки, `admin` или суперадмин |
| POST | `/calls/guest-token/:token` | публичный (пре-JWT) | Гостевой JWT. Гейт модуля fail-closed (`settings.modules.calls !== true` → 403). 404 `LINK_NOT_FOUND`, 410 `LINK_EXPIRED`, 410 `LINK_EXHAUSTED` (инкремент `uses` под охраной `WHERE` — гонка не уводит за `max_uses`). Ответ: JWT 1h `{sub: отрицательный id, guest: true, guestName, db}` + поле `turn` при настроенном `TURN_SHARED_SECRET` |

### voice-command

Префикс «Агент,» срезается. Встроенные: **«статус»** — запуски автоматизаций за последний час; **«протокол»** — подсказка про панель транскрипции. Остальное — матч по **всем** словам (после срезки «запусти/запуск/выполни»; слова >3 симв., стемминг конечной гласной) в имени активной автоматизации: одно совпадение → запуск (`runAutomationManually`); несколько → «Уточните: …» (до 5 вариантов); ноль → «Не нашёл…».

## WS channel `calls`

Подключён в `api/v2/ws.js` через `case 'calls':`; гасится только при явном `modules.calls === false`.

Гостю (guest JWT) разрешены только WS-типы `auth | calls | ping`, остальное → `GUEST_FORBIDDEN`; внутри канала действия `call.*` гостю отвергаются (`GUEST_FORBIDDEN`) — гость участвует только в комнатных `room.*`/`peer.*`.

### Direct (1-1) — только члены

- `call.invite { targetUserId }` → создаёт `calls_history` (status=pending), цели уходит `{callId, fromUserId, fromUserName}`; инициатору — `call.invite.ack { callId, targetUserId }`
- `call.accept|reject { callId }` — только цель, только из `pending` → `active|rejected`
- `call.cancel { callId }` — только инициатор → `cancelled`
- `call.timeout` — сервер → обоим через 30 с; статус `missed` + уведомление «Пропущенный звонок»
- `peer.offer|answer|ice|bye { callId, sdp|candidate }` — релей другой стороне; `peer.bye` из `active` → `completed`

### Room (комната, лимит `ROOM_LIMIT` = 12)

- `room.join { withVideo, withAudio }` — 13-й получает `room.full { limit: 12 }`; на комнату одна запись `calls_history` (kind=room, status=active)
- `peer.connect { peerUserId, peerUserName, polite }` — новичку по каждому присутствующему и остальным про новичка; `polite = мой id < peerUserId`
- `peer.offer|answer|ice|bye` без `callId` — комнатный путь: код читает **`targetUserId`** (`peerUserId` — легаси-алиас), в релей кладётся `peerUserId` отправителя
- `room.leave` / закрытие WS — остальным `peer.bye { peerUserId }`; последний вышел → `completed`
- `room.presence { users: [{userId, userName, joinedAt, withVideo, withAudio}] }` — броадкаст воркспейсу; guest-участник виден под своим `guestName`

## Disconnect Cleanup

По обрыву WS незавершённые прямые звонки чистятся по роли:
- **Инициатор оборвался** → `cancelled`, цели уходит `call.cancel`
- **Цель оборвалась** → `missed`, инициатору `call.timeout` + «Пропущенный звонок»

Комнатные membership чистятся (`peer.bye` остальным, обновление presence).

## Статусы

Достижимые: `pending` → `active | rejected | cancelled | missed`; `active` → `completed`.
`failed` числится в TERMINAL-наборе `service.js`, но ни один writer его не ставит — **недостижим**.

## Data model

Satellite-таблицы, per workspace:

- `calls_history(id, kind direct|room, initiator_user_id, target_user_id, started_at, ended_at, status, duration_sec)`
- `calls_participants(id, call_id, user_id, joined_at, left_at)`
- `calls_links(token PK, room_name, created_by, created_at, expires_at, max_uses, uses, revoked)`

## Env vars

- `TURN_SHARED_SECRET` — **опционален**; нет → endpoint отдаёт пустые креды, гостевой ответ без поля `turn` (STUN-фолбэк фронта)
- `TURN_URLS` (через запятую)
- `TURN_TTL_SEC` (default 3600)

Гайд по coturn: `docs/ops/coturn-setup.md` (путь от корня репозитория).
