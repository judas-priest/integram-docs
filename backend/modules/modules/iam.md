# Module: iam

**Path:** `src/api/v2/modules/iam/`
**Files:** `router.js`, `service.js`, `schema.js`
**Base URL:** `/api/v2/iam/...`  (no `:db` — global)
**Auth:** Most endpoints are public (rate-limited). Protected endpoints require JWT.

## Purpose

Global identity and access management. Handles user registration, login, JWT access tokens, refresh token rotation, password management, and magic link (auto-login) tokens. The only module that issues JWTs.

## Endpoints

| Method | Path | Auth | Rate Limit | Description |
|--------|------|------|------------|-------------|
| POST | `/iam/register` | None | 5/min/IP | Register new user |
| POST | `/iam/login` | None | 5/min/IP | Login → access token + refresh cookie |
| POST | `/iam/refresh` | Cookie | 10/min/IP | Refresh access token (reads HttpOnly cookie) |
| POST | `/iam/logout` | Cookie | None | Logout — clears refresh cookie |
| GET | `/iam/me` | JWT | — | Get current user profile |
| PATCH | `/iam/me` | JWT | — | Update name |
| POST | `/iam/change-password` | JWT | — | Change password (requires current password) |
| POST | `/iam/forgot-password` | None | 5/min/IP | Send password reset email |
| POST | `/iam/reset-password` | None | 5/min/IP | Confirm reset with token |
| GET | `/iam/auto-login/:token` | None | 5/min/IP | Magic link login |
| POST | `/iam/create-auto-login` | JWT | — | Generate magic link token |

## Token Architecture

**Access token** — short-lived JWT (15 minutes). Lives in memory only (`globalToken` in frontend store). Never stored in localStorage or cookies.

**Refresh token** — long-lived (30 days). Stored server-side in `_v2_refresh_tokens` table AND sent as an `HttpOnly; Secure; SameSite=Strict` cookie at path `/api/v2/iam`. Never accessible to JavaScript.

**Token rotation** — each refresh issues a new RT and invalidates the old one. Reuse of an old RT triggers family nuke (all RTs for the user are deleted) unless it happened within the 30-second grace window (multi-tab race condition protection).

## Refresh Cookie

```
Set-Cookie: refresh_token=<value>; HttpOnly; Secure; SameSite=Strict;
            Path=/api/v2/iam; Max-Age=2592000
```

Only sent to `/api/v2/iam` — no cookie leakage to workspace endpoints.

## Rate Limiting

Auth endpoints: 5 attempts per minute per IP. Localhost IPs are exempt (dev/test). Uses `utils/rate-limit.js` with `keyFn: (req) => 'auth:' + req.ip`.

## Magic Link (Auto-Login)

`POST /iam/create-auto-login` generates a signed JWT with short TTL. `GET /iam/auto-login/:token` validates it and issues a full session. Used for demo/onboarding flows.

## Zod Schemas (`schema.js`)

`registerSchema`, `loginSchema`, `refreshSchema`, `logoutSchema`, `updateMeSchema`, `changePasswordSchema`, `forgotPasswordSchema`, `resetPasswordSchema`

## DB Tables (global public schema)

- `_v2_users` — `id`, `email`, `name`, `username`, `password_hash`, `avatar_url`, `is_superadmin`, `is_bot`, `created_at`, `updated_at`
- `_v2_refresh_tokens` — `id`, `user_id`, `token_hash`, `family` (UUID), `created_at`, `expires_at`
- `_v2_password_reset_tokens` — `id`, `user_id`, `token_hash`, `expires_at`, `used_at`
