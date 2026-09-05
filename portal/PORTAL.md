# Portal — Документация

**Стек:** Nuxt 3, Vue 3
**Директория:** `portal/`
**Dev-сервер:** `http://localhost:3000` (proxied через backend на `/:db/portal/*`)

Клиентский портал — публичный сайт с каталогом, корзиной, заказами, личным кабинетом, KB, AI-чатом и аналитикой. Создаётся из workspace в один клик через визуальный редактор.

## Содержание

- [Архитектура](#архитектура)
- [Структура директорий](#структура-директорий)
- [Страницы](#страницы)
- [Модули страниц](#модули-страниц)
- [Конфигурация](#конфигурация)
- [Визуальный редактор](#визуальный-редактор)
- [Аутентификация (OTP)](#аутентификация-otp)
- [Роли клиентов](#роли-клиентов)
- [Корзина и заказы](#корзина-и-заказы)
- [AI-чат](#ai-чат)
- [Knowledge Base](#knowledge-base)
- [Аналитика](#аналитика)
- [SEO](#seo)
- [Шаблоны](#шаблоны)
- [Зависимость от EAV-таблиц](#зависимость-от-eav-таблиц)
- [Coding Patterns](#coding-patterns)
- [API](#api)

---

## Архитектура

```
Клиент (браузер)
       │
       ▼
┌─────────────────────────────┐
│   Nuxt 3 SSR (:3000)       │
│                             │
│   pages/[db]/portal/...     │ ← SSR рендер + hydration
│   composables/              │ ← auth, cart, config, chat, analytics
│   layouts/portal.vue        │ ← topbar + footer + chat widget
│                             │
│   Proxy: /api/v2/* → :8081  │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│   Express Backend (:8081)   │
│                             │
│   /:db/portal/api/config    │ ← конфигурация
│   /:db/portal/api/catalog   │ ← каталог (EAV)
│   /:db/portal/api/auth/*    │ ← OTP (Telegram, SMS)
│   /:db/portal/api/cart/*    │ ← корзина (EAV)
│   /:db/portal/api/orders    │ ← заказы (EAV)
│   /:db/portal/api/chat/*    │ ← AI-чат (LLM + tools)
│   /:db/portal/api/analytics/stats │ ← события
│   /:db/portal/*             │ ← proxy → Nuxt :3000
│                             │
│   _v2_portal_config (JSONB) │ ← конфиг в PostgreSQL
│   _v2_portal_events         │ ← аналитика (TimescaleDB)
└─────────────────────────────┘
```

**Один Nuxt-инстанс** обслуживает все воркспейсы. Изоляция — через `[db]` в URL: `/:db/portal/*`.

---

## Структура директорий

```
portal/
├── pages/          — Nuxt-страницы (файловая маршрутизация)
├── components/     — Переиспользуемые Vue-компоненты
├── composables/    — Nuxt Composables (useXxx)
├── layouts/        — Nuxt layouts
├── server/
│   └── middleware/ — Nitro SSR middleware (portal-auth.ts)
├── app.vue         — Корневой компонент
└── nuxt.config.ts  — Конфигурация Nuxt
```

Подробная документация:

| Раздел | Файл |
|--------|------|
| [Pages](pages.md) | страницы портала |
| [Components](components.md) | компоненты |
| [Composables](composables.md) | composables |

---

## Страницы

| Роут | Страница | Auth | Описание |
|------|----------|------|----------|
| `/:db/portal` | Home | нет | Модульная главная: landing, hero, каталог, featured, gallery, about |
| `/:db/portal/catalog` | Каталог | нет | Сетка товаров с hero-блоком |
| `/:db/portal/catalog/:slug` | Карточка товара | нет | Фото, цена, описание, корзина/wishlist. Schema.org Product JSON-LD |
| `/:db/portal/cart` | Корзина | нет | Кол-во, итого, guest checkout modal |
| `/:db/portal/orders` | Заказы | да | Список заказов клиента |
| `/:db/portal/orders/:id` | Детали заказа | да | Статус, дата, позиции, трекинг |
| `/:db/portal/profile` | Профиль | да | Редактируемые поля из EAV |
| `/:db/portal/documents` | Документы | да | Фильтрация: тип, статус, даты, поиск |
| `/:db/portal/metrics` | Метрики | да | KPI-тайлы + line charts (Chart.js), период 7d/30d/90d/1y |
| `/:db/portal/kb` | Knowledge Base | нет | Поиск (vector-powered), категории |
| `/:db/portal/kb/:id` | Статья KB | нет | Тело HTML + related articles (vector similarity) |
| `/:db/portal/support` | Поддержка | да | Список тикетов + создание нового |
| `/:db/portal/gallery` | Галерея | нет | CSS-grid изображений через NuxtImg |
| `/:db/portal/contacts` | Контакты | нет | Адрес, телефон, email, часы, карта |
| `/:db/portal/wishlist` | Избранное | нет | localStorage, без серверной синхронизации |
| `/:db/portal/staff` | Staff панель | staff | Управление заказами, клиентами для сотрудников |
| `/:db/portal/teamchat` | Командный чат | да | Teamchat воркспейса внутри портала |
| `/:db/portal/decisions/:id` | Решения | да | Архитектурные решения воркспейса |
| `/:db/portal/meta-kb/:topicId` | Meta KB | да | Дебатная база знаний |
| `/:db/portal/auth` | Вход | нет | Двухшаговый OTP: Telegram или SMS-код |
| `/:db/portal/coming-soon` | Заглушка | нет | Показывается при `active === false` |

---

## Модули страниц

Каждая страница состоит из модулей. Модули — визуальные блоки с собственным конфигом и привязкой к EAV-данным.

### Модули главной страницы

| Тип | Описание |
|-----|----------|
| `landing` | Hero-баннер с CTA |
| `hero` | Полноширинный баннер с изображением |
| `catalog` | Сетка товаров из EAV |
| `featured_products` | Выделенные товары |
| `gallery` | Фотогалерея |
| `about` | Блок «О компании» |
| `reports` | Виджет отчёта |
| `auth` | Блок входа |

### Универсальные модули (любая страница)

| Тип | Описание |
|-----|----------|
| `text` | Произвольный HTML-блок |
| `banner` | Полноширинное изображение с ссылкой |
| `faq` | Аккордеон вопросов-ответов |
| `report_widget` | Таблица отчёта по ID |
| `mini_catalog` | Мини-каталог с фильтром по категории |
| `custom_code` | Произвольный Vue 3 SFC из codespace-репозитория, компилируется в рантайме. Полный доступ к API данных (`useObjects`, `useReport`, write-операции), CDN-библиотеки, shared state между модулями. Подробно: `docs/custom-code.md` |

### Конфиг модуля

```js
{
  _id: "uuid",           // уникальный ID (генерируется)
  type: "catalog",       // тип модуля
  visible: true,         // видимость
  locked: false,         // locked = основной модуль страницы
  roles: ["VIP"],        // доступ по ролям (опционально)
  settings: { ... },     // визуальные настройки (заголовки, высоты)
  config: {              // привязка к данным
    typeId: 42,          // EAV-тип (таблица)
    nameReqId: 100,      // колонка «название»
    priceReqId: 101,     // колонка «цена»
    imageReqId: 102,     // колонка «фото»
    // ...
  }
}
```

---

## Конфигурация

Весь конфиг портала — JSONB в таблице `_v2_portal_config` (одна строка на workspace).
Composable `usePortalConfig` загружает его при инициализации (SSR, `useAsyncData`).

### Branding

```js
{
  name: "Мой магазин",
  logo: "/api/v2/db/files/123",
  favicon: "/api/v2/db/files/124",
  description: "...",
  template: "natural",       // пресет: natural | premium | industrial | minimal
  primaryColor: "#6B7C3E",   // переопределение цвета
  bgColor: "#FAFAF7",
  surfaceColor: "#FFFFFF",
  textColor: "#2D2D2D",
  borderRadius: "medium",    // none | small | medium | large
  cardStyle: "bordered",     // bordered | shadow | flat
  spacing: "M",              // S | M | L
  contentWidth: "standard",  // narrow | standard | wide
  font: "DM Sans"            // Google Font
}
```

### 4 встроенных пресета

| Пресет | Шрифт | Primary | Стиль |
|--------|-------|---------|-------|
| `natural` | DM Serif Display / DM Sans | #6B7C3E (olive) | large cards, стандартная ширина |
| `premium` | Playfair Display / Source Sans 3 | #1A1A2E (dark navy) | shadow cards |
| `industrial` | Montserrat / Open Sans | #1C4E8A (blue-grey) | bordered cards, wide |
| `minimal` | Inter | #2563EB (blue) | flat cards |

### Топбар

```js
{
  phone: "+7 999 123-45-67",
  showSearch: true,
  logo: { url: "...", maxHeight: 40 },
  style: { position: "sticky", bg: "#fff", textColor: "#333" }
}
```

### Чат

```js
{
  enabled: true,
  botName: "Ассистент",
  greeting: "Здравствуйте! Чем помочь?",
  enabledForAnonymous: true,
  maxMessagesPerSession: 50,
  systemPromptExtra: "Ты консультант магазина...",
  suggestedQuestions: ["Как оформить заказ?", "Сроки доставки"],
  escalation: { enabled: true, responseTimeHint: "в течение часа" }
}
```

### Auth

```js
{
  clientsTypeId: 42,    // EAV-тип «Клиенты»
  phoneReqId: 100,      // колонка «Телефон»
  nameReqId: 101,       // колонка «Имя» (опц.)
  roleReqId: 102,       // колонка «Роль» (опц., ref на lookup)
  grantsConfig: { ... } // настройка грантов по ролям
}
```

### Analytics

```js
{
  yandexMetrikaId: "12345678",
  googleAnalyticsId: "G-XXXXXXXXXX"
}
```

---

## Визуальный редактор

Фронтенд-компонент `PortalVisualEditor.vue` + composable `usePortalEditor.js`.

### Три панели

1. **Левая** — дерево страниц и модулей. Drag-to-reorder, показать/скрыть, добавить/удалить
2. **Центральная** — preview портала (iframe)
3. **Правая** — контекстные настройки: branding, пресеты, конфиг выбранного модуля, роли доступа

### Возможности

- 50 шагов undo/redo
- Драфт в localStorage (автосохранение)
- 4 пресета стилей (natural, premium, industrial, minimal)
- Per-module конфиг: привязка к EAV-таблицам и колонкам
- Per-page и per-module контроль ролей
- Публикация: сохранение конфига + `active = true`

---

## Аутентификация (OTP)

Портальные клиенты — EAV-записи в таблице workspace (не системные пользователи).

### Telegram OTP (основной метод)

1. Клиент нажимает "Войти через Telegram" → `POST /auth/telegram/start`
2. Backend возвращает `sessionToken` + `telegramLink` (deep link на бота)
3. Клиент переходит в Telegram → бот авторизует сессию
4. Фронт поллит `GET /auth/telegram/status?token=...` каждые 2 сек
5. При `status === 'authorized'` — backend выдаёт Portal JWT, устанавливает httpOnly cookie `portal_jwt` (30 дней)
6. `mergeCart()` — гостевая корзина из localStorage сливается с серверной

**Автологин:** при наличии валидной `portal_jwt` cookie, SSR-запрос `/api/auth/me` возвращает профиль автоматически.

### Staff login (email + password)

Сотрудники рабочего пространства могут войти на портал через `POST /api/auth/staff/login`:

```json
{ "email": "manager@company.com", "password": "..." }
```

Возвращает `portal_jwt` с `user_type: 'portal_staff'` и TTL **30 суток** — столько же, сколько
у клиентского токена. Кука живёт ровно столько же, сколько токен.

**Срок настраивается переменной среды `PORTAL_STAFF_JWT_TTL`** (`30d`, `8h`, `45m`, голое число —
секунды). Без переменной — 30 суток; неразобранное значение отбрасывается с записью в журнал и тем же
умолчанием. Переменная меняет и срок токена, и `Max-Age` куки.

> До 18.06.2026 (коммит `e7ed9342c`) срок был 8 часов, и здесь два года было написано «8h» при
> `PORTAL_STAFF_JWT_TTL = '30d'` в коде.

Используйте `portalAuth('admin', 'owner', 'editor')` middleware на роутах, доступных только сотрудникам.

**Журнал.** Каждая неудачная попытка входа пишется `log.warn` с полями
`{ db, email, ip, reason }`, где `reason` — `unknown_email` | `bad_password` | `not_workspace_member` |
`not_employee`; успешный вход — `log.info` с `{ db, email, userId, ip }`. Пароль в журнал не попадает
ни в каком виде. Без записи успеха «тысяча неудач и тишина» неотличимо от «тысяча неудач и вход».

**Форма входа** — на странице `/:db/portal/auth`, кнопка «Вход для сотрудников» рядом с входом
через Telegram (`pages/[db]/portal/auth.vue`). До 12.08.2026 ручка существовала, а вызвать её было
неоткуда: в интерфейсе портала не было ни одной формы с почтой и паролем, и сотрудник с рабочей
учётной записью упирался в стену.

Запрос обязан идти с `credentials: 'include'` — без него браузер не примет куку, ответ будет 200,
а пользователь останется неавторизованным.

Claim `user_type` присутствует во всех portal JWT:
- `'portal_customer'` — клиент (OTP / Telegram-вход)
- `'portal_staff'` — сотрудник (email+password-вход)

### SMS OTP (подготовлен, отключён)

Код готов (`auth/request`, `auth/verify`), но закомментирован в `auth.vue` — SMS-провайдер не настроен.

### Rate limits

- auth/request: 5 req/мин на телефон
- auth/verify: 10 req/мин на IP
- orders/guest: 10 req/мин на IP

---

## Роли клиентов

Роли — обычная ref-колонка на EAV-записи клиента, указывающая на lookup-таблицу.

```js
auth: {
  roleReqId: 102,           // ref-колонка «Роль» на клиенте
  grantsConfig: {
    grantsTypeId: 50,        // дочерняя таблица грантов у роли
    targetReqId: 200,        // колонка «Тип» (на что грант)
    levelReqId: 201,         // колонка «Уровень» (READ/WRITE)
    maskReqId: 202,          // колонка «Маска полей» (опц.)
  }
}
```

- `pages[].roles: ["VIP"]` — страница видна только этим ролям
- `modules[].roles: ["VIP"]` — модуль виден только этим ролям
- Страницы/модули без `roles` — видны всем

---

## Корзина и заказы

### Корзина

- **Гость**: localStorage (`portal_cart_:db`), без серверной синхронизации
- **Авторизованный**: серверная корзина через EAV

При логине: `mergeCart()` → гостевые позиции пушатся на сервер, localStorage очищается.

### Guest orders

Заказ без регистрации: `POST /portal/api/orders/guest` — имя + телефон + позиции.
Создаёт EAV-объект заказа + дочерние позиции. Rate limit: 10 req/мин/IP.

### CDEK

Если `config.connectors.cdek === true`, GuestCheckoutModal показывает шаг расчёта доставки через `POST /portal/api/delivery/calculate`.

---

## AI-чат

Виджет в правом нижнем углу. Включён если `chat.enabled === true`.

### Двухуровневая маршрутизация

1. **Router agent** (быстрая модель) — классифицирует intent, маршрутизирует к sub-agent
2. **Sub-agents** (умная модель) — каждый со своими инструментами, до 8 раундов

### Sub-agents

| Агент | Auth | Инструменты |
|-------|------|-------------|
| `catalog` | нет | `catalog_search`, `catalog_item` |
| `cart` | нет | `cart_view`, `cart_add`, `cart_update`, `cart_remove` |
| `orders` | да | `order_list`, `order_status` |
| `kb` | нет | `kb_search`, `kb_article` |
| `support` | да | `ticket_create`, `ticket_list` |
| `documents` | да | `doc_list` |
| `profile` | да | `profile_view`, `profile_update` |
| `metrics` | да | `metrics_query` |

Набор агентов формируется динамически: только для включённых страниц + фильтр по ролям.

### Сессии

- Анонимные — по `sessionKey` (UUID в sessionStorage)
- Авторизованные — по `client_id`
- Контекст: последние 10 сообщений
- Rate limit: anonymous 10 msg/мин, auth 30 msg/мин

---

## Knowledge Base

KB — EAV-таблица workspace, подключённая через конфиг.

- Поиск: vector-powered (pgvector), semantic search по эмбеддингам
- Related articles: cosine similarity

---

## Аналитика

Таблица `_v2_portal_events` (TimescaleDB hypertable):

| Событие | Когда |
|---------|-------|
| `page_view` | Клиент открыл страницу |
| `product_view` | Клиент открыл карточку товара |
| `order_created` | Оформлен заказ |

Внешняя: Yandex Metrika и Google Analytics инжектируются из конфига в `app.vue`.

---

## SEO

Composable `usePortalSeo.js` генерирует:

- `<title>`, `<meta description>`, Open Graph, Twitter Card, Canonical URL
- Favicon из branding, ссылка на sitemap
- JSON-LD: Organization, WebSite, BreadcrumbList (авто из роута)
- Product JSON-LD на карточке товара (Schema.org Product)
- ItemList JSON-LD на странице каталога (`/catalog`) — до 100 первых товаров

Sitemap: `GET /portal/api/sitemap.xml` — каталог + публичные страницы из конфига (с `lastmod`/`changefreq`/`priority`).
SSR cache: home и catalog — `cache-control: public, max-age=5, stale-while-revalidate=30`.

---

## Шаблоны

При сохранении workspace как шаблона `mapPortalIds(config, realToLocal, 'extract')` заменяет числовые EAV-ID на портативные строки (`T_CLIENTS`, `C_CL_PHONE`). При создании из шаблона — обратная замена + `regenerateModuleIds()` + `active = false`.

| Шаблон | Slug | Описание |
|--------|------|----------|
| **Интернет-магазин** | `sys-ws-ecommerce` | 8 страниц: home, catalog, orders, cart, profile, documents, support, kb, contacts |
| **Производство** | `sys-ws-manufacturing` | home, parts catalog, incidents, kb, profile |

Конфиги: `backend/src/data/templates/workspaces/ecommerce.json`, `manufacturing.json`.

---

## Зависимость от EAV-таблиц

Портал **не хранит собственных данных** — все сущности (товары, заказы, клиенты, KB, тикеты, документы) — обычные EAV-записи workspace.

| Поле конфига | Что указывает |
|---|---|
| `catalog.typeId` | ID таблицы «Товары» |
| `catalog.priceReqId` | ID колонки «Цена» |
| `orders.typeId` | ID таблицы «Заказы» |
| `clients.typeId` | ID таблицы «Клиенты» |
| `kb.typeId` | ID таблицы «База знаний» |
| `support.typeId` | ID таблицы «Тикеты» |
| `metrics.reportIds` | ID отчётов для KPI |

Удаление EAV-таблицы/колонки сломает соответствующую страницу портала.

---

## CSP (Content Security Policy)

Портал выставляет CSP через `portal/server/middleware/csp.ts` (Nitro server middleware).

Ключевые директивы:
- `script-src 'unsafe-inline' 'unsafe-eval'` — оба обязательны: `unsafe-eval` для vue3-sfc-loader (`new Function()`), `unsafe-inline` для Nuxt 3 SSR-гидрации (`__NUXT_DATA__` inline scripts)
- `script-src https://esm.sh https://unpkg.com` — CDN для custom_code библиотек
- `connect-src https://esm.sh` — fetch() внутри vue3-sfc-loader getFile()

> Если убрать `unsafe-inline` из `script-src` — Nuxt SSR-гидрация сломается с `TypeError: 'target' argument of Proxy must be an object, got undefined`.

---

## Immersive Mode

Когда **все** модули на странице имеют тип `custom_code`, портал автоматически скрывает topbar (хедер, лого, навигацию) и футер — страница рендерится в полноэкранном режиме. Определяется computed-свойством в `layouts/portal.vue`. Используется для PLM shell и других полноэкранных кастомных приложений.

---

## Coding Patterns

### Конфигурация

- Все EAV-ID берутся из конфига — **никогда не хардкодить** `typeId`/`reqId`
- `"_value"` в конфиге — sentinel «использовать display name объекта» — не заменять реальным reqId
- EAV-запросы: `WHERE a.t = ? AND a.up = ?` оба = `typeId` (не `up = 1` — классический баг)

### Аутентификация

- Пользователи портала ≠ пользователи workspace. Не использовать `requireJwt`
- SSR-guard: `server/middleware/portal-auth.ts` (Nitro) проверяет `portal_jwt` cookie до рендера
- Для проверки авторизации — `usePortalAuth` вызывает `/api/auth/me`

### Глобальные стили

Глобальные стили вынесены в `assets/portal.css` (подключён через `nuxt.config.ts` `css:[]`). Причина: scoped styles в `[db]` route pages вызывали 500 при Firefox back-navigation.

### Nuxt config

- `ssr: true`
- `css: ['~/assets/portal.css']` — глобальные стили портала
- `nitro.devProxy` — проксирует `/api/v2` на `http://127.0.0.1:8081` в dev
- `runtimeConfig.apiBaseUrl` — server-only URL бэкенда для SSR-fetch
- `routeRules` — SSR-кеш на главной и каталоге

### Добавить новый модуль

1. Backend: обработчики в `portal/router.js` + `getModuleConfig`
2. Редактор: карточка модуля в `frontend/src/composables/usePortalEditor.js`
3. Nuxt: страница в `portal/pages/[db]/portal/`
4. Обновить `docs/pages.md` и `docs/components.md`

---

## API

### Публичные (без авторизации)

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/portal/api/config` | Конфигурация. По ролям **не** фильтруется — применяется только `sanitizePublicConfig` (вырезание секретов); `filterPagesByRole` работает лишь в `/portal/api/auth/me` |
| GET | `/portal/api/catalog` | Каталог (пагинация, `?category=` фильтр) |
| GET | `/portal/api/catalog/categories` | Список категорий для фильтра |
| GET | `/portal/api/catalog/:slug` | Карточка товара |
| GET | `/portal/api/sitemap.xml` | Sitemap XML |
| POST | `/portal/api/auth/request` | Запросить OTP (SMS или Telegram). Body: `{ phone }` |
| POST | `/portal/api/auth/verify` | Проверить OTP, установить httpOnly cookie `portal_jwt`. Body: `{ phone, code }` |
| POST | `/portal/api/auth/telegram/start` | Начать Telegram OTP |
| GET | `/portal/api/auth/telegram/status` | Статус Telegram OTP |
| POST | `/portal/api/auth/logout` | Выход |
| GET | `/portal/api/auth/me` | Текущая портальная сессия (`optionalPortalJwt`; без куки — `data: null`) |
| POST | `/portal/api/auth/staff/login` | Вход сотрудника (email+password, JWT 30d по умолчанию — см. `PORTAL_STAFF_JWT_TTL`, user_type:'portal_staff') |
| POST | `/portal/api/orders/guest` | Заказ без регистрации. Body: `{ name, phone, items: [{name, qty, price}] }` |
| POST | `/portal/api/analytics/event` | Записать событие |
| GET | `/portal/api/kb/articles` | Список статей KB |
| GET | `/portal/api/kb/articles/:articleId` | Статья KB |
| GET | `/portal/api/kb/search?q=` | Семантический поиск по KB |
| GET | `/portal/api/files/:filename` | Скачать файл по имени (`optionalPortalJwt`). Изображения (`.jpg/.jpeg/.png/.gif/.webp/.svg`) открыты анониму — на них стоит оформление портала. Звук и видео (`.webm/.ogg/.oga/.opus/.m4a/.mp3/.wav/.mp4/.mov`) требуют портальной сессии: без неё 401 `AUTH_REQUIRED`. Прочие расширения — 403 `FORBIDDEN` |

### Авторизованные (portal_jwt)

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/portal/api/orders` | Заказы клиента |
| GET | `/portal/api/orders/:id` | Детали заказа |
| GET/PUT | `/portal/api/profile` | Профиль клиента |
| GET | `/portal/api/documents` | Документы клиента |
| GET | `/portal/api/metrics` | Метрики клиента |
| GET | `/portal/api/cart` | Получить корзину |
| POST | `/portal/api/cart/items` | Добавить позицию. Body: `{ productId, qty }` |
| PUT | `/portal/api/cart/items/:itemId` | Изменить количество. Body: `{ qty }` |
| DELETE | `/portal/api/cart/items/:itemId` | Удалить позицию |
| DELETE | `/portal/api/cart` | Очистить корзину |
| POST | `/portal/api/cart/merge` | Слить гостевую корзину |
| POST | `/portal/api/chat/message` | Сообщение в чат |
| GET | `/portal/api/chat/history` | История чата |
| POST | `/portal/api/chat/feedback` | Feedback на сообщение |

### Staff (portal_staff JWT)

| Метод | Путь | Описание |
|-------|------|----------|
| POST | `/portal/api/staff/dadata/suggest/:type` | DaData proxy: address, party, bank, fio, email. Body: `{ query, count? }` |
| POST | `/portal/api/staff/dadata/findById/:type` | DaData findById: party, address, bank. Body: `{ query }` |
| POST | `/portal/api/fn/:repo/:name` | Выполнить server function из codespace. Body: произвольный JSON → args функции |

### EAV-таблицы (`optionalPortalJwt` + гранты)

Заслон здесь **не** staff-JWT. Чтение (`/tables/...`) стоит под `optionalPortalJwt` — аноним допускается, а решает проверка гранта на `typeId` (`portalGrantsMiddleware` + `checkGrant`). Запись требует портальную куку (`requirePortalJwt`) и грант WRITE. Отдельных маршрутов вида `/portal/api/tables/:typeId/objects/:id` не существует — ни на чтение одной записи, ни на запись; запись живёт на `/portal/api/objects`.

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/portal/api/tables/:typeId/schema` | Схема колонок таблицы. 60/мин |
| GET | `/portal/api/tables/:typeId/objects` | Список записей EAV-таблицы (с пагинацией, фильтрацией). 200/мин |
| POST | `/portal/api/objects` | Создать запись (portal_jwt + грант WRITE на `typeId`). Body: `{ typeId, name, fields, parentId? }`. 30/мин |
| PATCH | `/portal/api/objects/:id` | Обновить поля записи (portal_jwt + грант WRITE). 60/мин |
| DELETE | `/portal/api/objects/:id` | Удалить запись (portal_jwt). 30/мин |

**Список записей — фильтр по родителю.** `GET /portal/api/tables/:typeId/objects` всегда отбирает записи одного родителя: `?parentId=N` — записи под записью N, без параметра — верхний уровень (корень воркспейса). Поэтому дочерняя таблица без `parentId` отдаёт `total: 0`. Ответ всегда содержит применённый родитель и его происхождение:

```json
{ "ok": true, "data": { "items": [], "total": 0, "parentId": 1, "parentScope": "default_root" } }
```

`parentScope`: `explicit` — родителя задал вызывающий, `default_root` — подставлен корень. Пустой ответ с `default_root` означает «под корнем ничего нет», а не «таблица пуста». `parentId=0` и нечисловое значение → 400 `INVALID_PARENT_ID`.

### Админские (workspace JWT + admin)

| Метод | Путь | Описание |
|-------|------|----------|
| POST | `/portal/api/config` | Сохранить конфиг |
| PUT | `/portal/api/config/active` | Опубликовать / снять с публикации. Body: `{ active: true|false }` |
| GET | `/portal/api/config/full` | Сырой конфиг вместе с секретами (визуальный редактор грузится только им) |
| GET | `/portal/api/config/history` | Последние снимки конфига (`?limit=` 1–100, по умолчанию 20). Несут секреты — админская поверхность наравне с `/config/full` |
| POST | `/portal/api/config/restore` | Откат конфига к снимку. Body: `{ historyId }`. Через `upsertConfig` (заменённый конфиг сам попадает в историю); ref-валидация сознательно не проходит |
| GET | `/portal/api/analytics/stats` | Статистика событий (1d/7d/30d) |
| POST | `/portal/api/legal/generate` | AI-генерация юридических документов |
| GET | `/portal/api/reports/:reportId` | Данные отчёта (только отчёты из portal config, иначе 403). **Заслон не админский:** маршрут идёт под `optionalPortalJwt` + `portalGrantsMiddleware`, то есть достижим анонимно, а решает грант и перечень отчётов в конфиге |
| GET | `/portal/api/bots` | Список Telegram-ботов воркспейса |
| POST | `/portal/api/bots` | Создать бота. Body: `{ name, username, token, config? }` |
| GET | `/portal/api/bots/:id` | Получить бота |
| PATCH | `/portal/api/bots/:id` | Обновить бота. Body: `{ name?, username?, token?, enabled?, config? }` |
| DELETE | `/portal/api/bots/:id` | Удалить бота |
| POST | `/portal/api/bots/:id/sync` | Синхронизировать конфиг бота в Telegram API (команды, описание, меню) |
| GET | `/portal/api/bots/:id/status` | Статус бота из Telegram API (getMe + getWebhookInfo) |
| POST | `/portal/api/bots/:id/test` | Отправить тестовое сообщение. Body: `{ chatId, text }` |

Bot config schema: `{ description?, shortDescription?, welcomeMessage?, menuButton?: {type}, employeeTable?: {typeId, chatIdReqId, roleReqId} }`.
Bot reactions (commands + keywords) stored as automations with `trigger.botId` — use `GET /automations?botId=N` to list.

### Глобальный

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/portal/check-domain` | Проверка кастомного домена (для Caddy) |

## AI Agent API

Portal custom_code components can call the AI orchestrator via `props.api.runAgent()`.

Enable via portal config `agent.enabled: true`. Configure tool permissions per role in `agent.tools` and `agent.roleOverrides`. See `backend/docs/modules/portal.md` for full config schema.

### SSE Events

When `runAgent()` runs, the stream yields:

| Event | Fields | Description |
|-------|--------|-------------|
| `RUN_STARTED` | none | Orchestrator initialized |
| `TEXT_MESSAGE_CONTENT` | `content: string` | LLM streamed text (accumulate for full response) |
| `TOOL_CALL_START` | `tool: string, args: {...}` | Tool invocation began |
| `TOOL_CALL_END` | `tool: string, result: {...}` | Tool completed, returned result |
| `STATE_DELTA` | `delta: JSONPatch[]` | Memory state changed (paths: `/memories/{key}`, `/shared/{key}`) |
| `RUN_FINISHED` | none | Run completed |
| `error` | `message: string` | Error occurred |

### STATE_DELTA Example (custom_code)

```js
for await (const event of props.api.runAgent('Analyze client history')) {
  if (event.type === 'STATE_DELTA') {
    // event.delta is JSON Patch format
    // Example: [{ op: 'add', path: '/memories/client_segment', value: 'VIP' }]
    console.log('Memory updated:', event.delta);
  }
  if (event.type === 'TEXT_MESSAGE_CONTENT') {
    output.value += event.content;
  }
  if (event.type === 'RUN_FINISHED') break;
}
```

### Swarm Memory Integration

Portal chat and orchestrator integrate swarm memory:
- Before LLM executes: relevant memories injected into system prompt via `recall()`
- After run completes: insights auto-extracted and stored via `extractSessionInsights()`
- Accessible to all agents via cross-session memory layer

See `backend/docs/modules/portal.md` section "Swarm Memory Integration (Chat)" for details.
