# ADR-006: Portal JWT отдельно от workspace JWT

**Статус:** accepted

## Контекст

Портал — публичный сайт для клиентов компании. Клиенты не являются пользователями workspace и не должны иметь доступ к API workspace. При этом и портал, и workspace работают на одном backend.

## Решение

Два полностью изолированных слоя аутентификации:

**Workspace JWT** — для сотрудников:
- Выдаётся через `POST /api/v2/iam/login`
- Access token в памяти, refresh token в `refresh_token` HttpOnly cookie
- Middleware: `jwtAuth` → `requireJwt` → `workspaceRoleMiddleware`
- Secret: `JWT_SECRET`

**Portal JWT** — для клиентов портала:
- Выдаётся через OTP (Telegram или SMS) или staff login (email+password, 8h)
- Хранится в `portal_jwt` HttpOnly cookie, TTL 30 дней (клиент) / 8 часов (сотрудник)
- Payload: `{ db, clientObjectId, user_type: 'portal_customer'|'portal_staff' }`
- Middleware: `requirePortalJwt` / `requireStaffPortalJwt` / `optionalPortalJwt`
- Secret: `PORTAL_JWT_SECRET` (отдельная env-переменная!)
- SSR-guard: `portal/server/middleware/portal-auth.ts` — проверяет наличие cookie до рендера

Клиенты портала — EAV-записи в workspace-таблице (`clients.typeId` из конфига), не системные пользователи.

## Последствия

- `requireJwt` на portal-роутах запрещён — он для workspace
- `clientObjectId` в portal JWT — это ID объекта в EAV, не `_v2_users.id`
- `PORTAL_JWT_SECRET` ≠ `JWT_SECRET` — разные env-переменные, должны быть разными значениями
- Конфиг портала (`_v2_portal_config`) хранит `clientsTypeId`, `phoneReqId` и т.д. — все EAV-ID из него, не хардкодить
- `"_value"` в конфиге = sentinel «использовать display name объекта», не заменять реальным reqId
