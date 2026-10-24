# Module: forms

**Path:** `src/api/v2/modules/forms/`
**Files:** `router.js`, `service.js`
**Base URL:** `/api/v2/:db/forms/...`
**Auth:** `GET /:token` is public (no auth). Management endpoints require JWT + `admin`.

## Purpose

Public data collection forms. Each form is linked to a table (`typeId`) and generates a unique token. Anyone with the token URL can submit records to the linked table without logging in.

## Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/forms/:token` | Public | Get form schema (columns, config) by token |
| GET | `/forms` | JWT | List forms for this workspace |
| POST | `/forms` | JWT + admin | Create form |
| PUT | `/forms/:token` | JWT + admin | Update form config/settings |
| DELETE | `/forms/:token` | JWT + admin | Delete form |

## Form Schema

```json
{
  "typeId": 123,
  "config": {
    "title": "Contact Us",
    "fields": [42, 43, 44],
    "submitLabel": "Send",
    "successMessage": "Thank you!"
  },
  "parentId": 1,
  "expiresAt": "2026-12-31T23:59:59Z"
}
```

- `config.fields`: array of reqTypeIds to show in the form
- `config.consent`: optional 152-ФЗ consent block `{ "text": "...", "version": "1" }` (см. «Согласие на обработку ПД»)
- `parentId`: EAV parent node for submitted records (usually root = 1)
- `expiresAt`: optional expiry; submissions to expired forms are rejected

## Submission Flow

Form submission (creating an object) goes through the standard objects API authenticated with the form token — not via this module directly. The form token authenticates the submission as a "form submitter" role.

Triggers `on_form_submit` automations after successful submission — after the `onSubmit` handler, if the form has one (see below).

## Согласие на обработку ПД (152-ФЗ, 02.10.2026)

Если в конфиге формы объявлен `consent: {text, version}`:

- `GET /forms/:token` (и портал-обёртка) возвращает блок `consent` — UI (`frontend/src/views/forms/FormPublic.vue`) рисует чекбокс с текстом и требует его перед отправкой.
- `submitForm` **отвергает** отправку без `body.consent === true` (400 «Требуется согласие…»). Проверка живёт внутри `submitForm` — публичная дверь (`api/v2/index.js`) и портал (`submitFormForPortal`, маршрут `portal/api/forms/:token/submit`) закрыты одной точкой; IP передаётся обоими вызовами.
- Факт согласия фиксируется в глобальной таблице `_v2_form_consents` (`token, db, object_id, ip, text_version, created_at`), заведение — `ensureConsentsTable`. Таблица внесена в реестр переноса (`workspace-carry`, `clone: skip` — token у клона перевыпускается) и каталоги уборки.

Пустой `config.consent` или его отсутствие — прежнее поведение: чекбокса нет, факт не пишется.

## Обработчик отправки — `config.onSubmit` (19.10.2026)

Серверная функция репозитория, которую ядро зовёт сразу после записи формы, в том же
запросе; её ответ отправитель получает в ответе формы. Нужна, когда отправителю надо
сразу вернуть то, что знает только сервер: ссылку на оплату, номер брони, итог с доставкой.
Автоматизация `on_form_submit` для этого не годится — она идёт после ответа, а сам аноним
функции репозитория звать не может (`portal/api/fn` — только сотрудники).

```json
{ "onSubmit": { "repo": "portal-components", "fn": "site-checkout", "args": { "slug": "shop-staging" } } }
```

- **Кто задаёт.** Только admin воркспейса (маршруты форм — `requireRole('admin')`). При
  создании и правке формы функция `api/<fn>.js` ищется в репозитории — нет такой: 400.
- **Как зовётся.** `executeServerFunction(pool, db, repo, fn, args, user)`; `args` — постоянные
  `onSubmit.args` (плоский объект строк, чисел, да/нет, ≤ 2 КБ — как `args` у
  `run_server_function`: одна функция на несколько форм или окружений) плюс
  `{ objectId, typeId, parentId, value }` записанной строки (эти имена в `args` заняты;
  читать строку — `getRecord(objectId)`);
  идентичность — `public_form` с повышением до admin, `elevated: true, reason: 'form_handler'`
  (как у `run_server_function`, TD-070). Секретов воркспейса нет — вызов приходит снаружи
  (та же граница, что у `portal/api/fn`, PM-890). Бюджеты — профиль `serverFunction`; таймаут —
  профиль, зажатый потолком анонимного запроса (`HANDLER_TIMEOUT_CAP_MS`, 15 с): заголовок
  `// timeout: N` в функции длиннее не работает — 180 с годятся очереди автоматизаций, не
  публичной отправке формы. Идемпотентность внешнего действия (создание платежа и т.п.)
  обязана жить в самой функции: реплей `X-Idempotency-Key` держится только 30 с.
- **Что уходит наружу.** Только `result.public` (≤ 8 КБ JSON): ответ формы —
  `data.handler = { ok: true, public }`. Без поля `public` — `{ ok: true }`, остальное, что
  вернула функция, отправитель не видит.
- **Сбой.** Запись остаётся: форма отвечает 201 с `handler: { ok: false, error }`, где error —
  `HANDLER_FAILED`, `HANDLER_TIMEOUT` или `HANDLER_RESULT_TOO_LARGE`; причина — в логе
  (`[forms] onSubmit handler failed`), не в ответе.
- **Порядок.** `object.created` эмитится после обработчика: `on_form_submit`-автоматизации
  видят запись, уже разобранную функцией, — разбор не идёт наперегонки в двух местах.
- **Ссылка-вход** (`createLoginLink`, способность `portalLink`) обработчику недоступна:
  отправитель формы ничем не доказал, что он — клиент, найденный по его данным. Выписывает
  её автоматизация и шлёт в проверенный канал (SMS, Telegram).
- **Хранение.** `onSubmit` живёт в `config`; публичное чтение формы (`GET /forms/:token`) его
  не показывает. `PUT` с конфигом без ключа `onSubmit` обработчик не снимает (конструктор
  форм собирает конфиг заново из своих полей), снять — `onSubmit: null`.
- **Повтор.** `X-Idempotency-Key` публичной отправки отдаёт сохранённый ответ целиком —
  с тем же `handler`, функция второй раз не зовётся.

## Idempotency

`X-Idempotency-Key` header supported on POST, cached for 30 seconds.

## DB Tables

- `_v2_forms` (global public schema) — `id`, `workspace_id`, `type_id`, `token` (UUID), `config` (JSONB), `parent_id`, `expires_at`, `created_by`, `created_at`
- `_v2_form_consents` (global, 02.10.2026) — `token`, `db`, `object_id`, `ip`, `text_version`, `created_at`: факт согласия 152-ФЗ
