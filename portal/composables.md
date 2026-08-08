# Composables

**Директория:** `composables/`

| Файл | Назначение |
|------|-----------|
| `useSanitize.js` | DOMPurify-обёртки для безопасного `v-html`. Экспортирует `sanitizeHtml(dirty)` — стандартная санитизация (разрешены `iframe`, `target`, `allowfullscreen`) и `sanitizeMapEmbed(dirty)` — только `<iframe>` с атрибутами для карт. Работает и в SSR (isomorphic-dompurify), и на клиенте. |
| `usePortalConfig.js` | Загружает конфиг портала через SSR `useAsyncData`. Возвращает `config`, `branding`, `seo`, `themeStyle` (CSS-переменные), `isActive`, `getPage`. Подгружает Google Fonts под текущий шаблон. Дополнительно возвращает `status` (AsyncData статус: `idle|pending|success|error`) и `error` (ошибка загрузки или `null`). |
| `usePortalAuth.js` | Состояние авторизации портала: гидрируется через SSR-запрос `/api/auth/me` (пробрасывает httpOnly cookie). Предоставляет `client`, `isAuthenticated`, `logout`, `refreshAuth`. Токен в localStorage не хранится — используется httpOnly cookie `portal_jwt`. Дополнительно возвращает `authStatus` (AsyncData статус) и `db` (computed из route.params). |
| `usePortalData.js` | Экспортирует `useCatalog(db, {page, limit, category})`, `useCategories(db)` и `useCatalogItem(db, slug)` — SSR-fetch данных каталога из `/api/v2/{db}/portal/api/catalog`. `category` — реактивный ref (ID категории), при изменении каталог перезапрашивается через `watch`. `useCategories` возвращает список категорий для фильтра. |
| `useCart.js` | Корзина: add, remove, update qty, merge (guest → auth), clear. Хранит guest-корзину в localStorage. |
| `useWishlist.js` | Избранное: `has`, `toggle`, `remove`, `count`. Хранится в localStorage (`portal_wish_{db}`). Нет серверной синхронизации — одинаково для гостей и авторизованных. |
| `usePortalAnalytics.js` | Отправка аналитических событий на `POST /api/v2/{db}/portal/api/analytics/event`. Методы: `trackPageView`, `trackProductView`, `trackOrderCreated`. Client-side only. |
| `usePortalSeo.js` | Устанавливает полный набор meta-тегов (og:*, twitter:*, canonical, JSON-LD: Organization/WebSite/BreadcrumbList). Принимает объект или функцию (для реактивных данных). |
| `usePortalChat.js` | AI-чат виджет. Отправка через `POST /portal/api/chat/message` ($fetch Nuxt, не SSE). Анонимные пользователи — UUID в sessionStorage (`portal_chat_session_key`). Возвращает: `messages` (история), `isOpen` (bool), `isLoading`, `error`, `sendMessage(text)`, `sendFeedback(messageIndex, feedback)`, `toggle()`, `close()`. |
| `usePortalDataLayer.js` | Singleton реактивный data layer для custom_code модулей. Все custom_code модули на странице разделяют один экземпляр. Предоставляет `fetchOnce` (кеш + дедупликация), `shared` (реактивное shared state), `on`/`emit` (pub/sub), `ensureUser` (кеш юзера). Использует plain Vue reactivity (не Nuxt composables) — работает внутри vue3-sfc-loader. |
| `usePortalTeamchat.js` | Teamchat: rooms, topics, messages, reactions, stars, polls, pins, receipts, reminders, file upload, agents. |
| `usePortalMetaKb.js` | Meta-KB: topics, debates, export (MD/DOCX), appropriation, analytics, iterations, changes. |
| `usePortalDecisions.js` | Decisions: list, detail, links, discussions, history, conflicts, iterations, KAG stats. |
| `useCustomCodeApi.js` | API-фасад для custom_code компонентов. Оборачивает DataLayer: `useCollection`, `useRecord`, `useReport`, `useDocuments` (read), `useObjects`, `useDocument`, `useTimeseries`, `useRelated` (extended data sources), `createRecord`, `updateRecord`, `runConnector` (write), `shared`, `on`/`emit`, `getUser`, `isAuthenticated`, `toast`. Биндинги парсятся из формата `table:123`/`report:456` в структурированные объекты. Передаётся через prop `api` в компилированный SFC. |

### `api.invokeAgents(slugs, task, params?)`

Параллельно вызывает несколько внешних агентов из реестра (fan-out паттерн по ADR-008).

**Параметры:**
- `slugs: string[]` — массив slug-ов агентов (максимум 10)
- `task: string` — задача для всех агентов
- `params?: object` — дополнительные параметры (например `{ context: { partId: 42 } }`)

**Возвращает:** `{ results: [{ slug, ok?, error?, result?, message? }], traceId: string }`

Graceful degradation: один упавший агент не блокирует остальных.

**Пример:**
```js
const { results } = await props.api.invokeAgents(
  ['design-agent', 'quality-agent'],
  `Analyze part ${partId}`,
  { context: { partId } }
);
// results[0] = { slug: 'design-agent', ok: true, result: {...} }
// results[1] = { slug: 'quality-agent', error: true, message: 'timeout' }
```

### runAgent(message, options?)

Async generator. Calls `POST /portal/api/agent/run` and yields SSE events.

Requires `agent.enabled: true` in portal config.

```js
const ctrl = new AbortController();
for await (const event of props.api.runAgent('Analyse order #42', { signal: ctrl.signal })) {
  if (event.type === 'TEXT_MESSAGE_CONTENT') output.value += event.content;
  if (event.type === 'TOOL_CALL_START') console.log(`Calling ${event.toolName}`);
  if (event.type === 'STATE_DELTA') console.log('Memory updated:', event.delta);
  if (event.type === 'RUN_FINISHED') break;
}
```

Options: `{ threadId?: string, agentSlug?: string, signal?: AbortSignal }`

Events yielded:
- `RUN_STARTED`
- `TEXT_MESSAGE_CONTENT` (content: string)
- `TOOL_CALL_START` (toolName: string, args: {...})
- `TOOL_CALL_END` (toolName: string, result: {...})
- `STATE_DELTA` (delta: JSONPatch[])
- `RUN_FINISHED`
- `error` (message: string)
