# Frontend — Документация

**Стек:** Vue 3, Vite, Pinia, Vue Router, PrimeVue, Tailwind CSS
**Директория:** `frontend/src/`
**Dev-сервер:** `http://localhost:5173`

## Структура директорий

```
frontend/src/
├── router/          — Vue Router: маршруты и guards
├── stores/          — Pinia: глобальное состояние
├── services/        — API-клиенты (по одному на модуль backend)
├── views/           — Страницы (один файл = один маршрут)
├── components/      — Переиспользуемые компоненты
├── composables/     — Vue Composables (useXxx)
├── layout/          — AppLayout, AppMenuItem, AppFooter
└── utils/           — Хелперы, константы, форматирование
```

Подробная документация по разделам:

| Раздел | Файл | Описание |
|--------|------|----------|
| [Router](router.md) | `router/index.js` | Маршруты, guards, module guard |
| [Stores](stores.md) | `stores/` | Pinia-хранилища |
| [Services](services.md) | `services/` | API-клиенты |
| [Views](views.md) | `views/` | Страницы по группам |
| [Components](components.md) | `components/` | Компоненты |
| [Composables](composables.md) | `composables/` | Composables |

## Архитектурные принципы

### Auth flow

1. Глобальный JWT (`globalToken`) в `stores/auth.js` — выдаётся при `/api/v2/iam/login`
2. Access token хранится в памяти (не localStorage). Refresh token — в HttpOnly cookie
3. При hard reload: `auth.iamMe()` восстанавливает пользователя из cookie
4. Workspace-scoped маршруты проверяют `meta.requiresAuth`, global — `meta.requiresGlobalAuth`

### Module guard

Некоторые маршруты закрыты, если workspace не включил модуль. Маппинг в `router/index.js` → `ROUTE_MODULE_MAP`. При неактивном модуле редирект на `dashboard`.

### API-клиенты (services/)

Каждый файл — тонкая обёртка вокруг `services/api.js`. Базовый URL: `/api/v2/{db}/...`. Авторизация: Bearer token из `stores/auth.js`.

### Состояние (stores/)

- `auth` — пользователь, JWT, логин/логаут
- `workspace` — список воркспейсов, настройки, включённые модули
- `ui` — Хлебные крошки навигации, заголовок страницы
- `notifications` — непрочитанные уведомления
- `aiChat` — состояние AI-чата
- `normalizer` — состояние нормализатора
- `calls` — P2P звонки и голосовая комната
- `teamchat` — командный чат: комнаты, топики, сообщения
- `decisions` — инженерные решения
- `pm` — управление проектами: задачи, спринты, доска

## Coding Patterns

### Слои — жёсткие правила

- **Services** (`services/*.js`) — все axios-вызовы только здесь. Компоненты и stores никогда не вызывают axios напрямую.
  - DB-scoped: `getDbApi(db).get('/path')`, global: `getGlobalApi().get('/path')`.
  - Возвращают `r.data` (распакованный из axios response).
- **Stores** (`stores/`) — только глобальное/кросс-страничное состояние (auth, workspace).
  - Локальное состояние страницы/компонента = `ref()` в компоненте, не в store.
  - Токены: refresh в HttpOnly cookie, access в памяти (`globalToken = ref(null)`), никогда в localStorage.
- **Views** (`views/`) — страницы. Использовать `<PageHeader>` для заголовков.

### Добавить новую страницу

1. `frontend/src/views/<module>/MyPage.vue` — компонент.
2. `frontend/src/router/index.js` — добавить маршрут.
3. Если требует авторизации — `meta: { requiresAuth: true }`.
4. Обновить `docs/router.md` и `docs/views.md`.

### Обработка ошибок

- Ошибки для пользователя → `toast.add({ severity: 'error', ... })`.
- Никогда не глотать ошибки (`catch {}`).

---

## Сборка и запуск

```bash
cd frontend
npm install
npm run dev      # dev-сервер на :5173
npm run build    # не запускать без необходимости
```

## Тесты

```bash
npm run test     # vitest
```

Тестовые файлы приняты в двух видах, оба живые:

- `*.test.js` рядом с тестируемым файлом — например `src/composables/usePointPresence.test.js`;
- подкаталог `__tests__/` рядом с тестируемыми файлами, внутри `*.spec.js` либо
  `*.test.js` — например `src/views/data/__tests__/object-view-point.spec.js`,
  `src/composables/__tests__/point-channel-has-client.test.js`.

## Как обновлять документацию

При добавлении нового маршрута → обновить `docs/router.md`.
При добавлении нового store → обновить `docs/stores.md`.
При добавлении нового сервиса → обновить `docs/services.md`.
При добавлении новой view → обновить `docs/views.md`.
