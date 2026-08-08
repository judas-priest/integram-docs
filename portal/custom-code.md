# Custom Code — кастомные Vue 3 компоненты

## Обзор

Модуль `custom_code` позволяет создавать произвольные Vue 3 SFC компоненты, которые рендерятся в портале. Код хранится в codespace-репозитории, компилируется в рантайме через vue3-sfc-loader.

Кастомный компонент — полноценный Vue 3 SFC (`<script setup>` + `<template>` + `<style scoped>`), который получает реактивный API-фасад для доступа к данным воркспейса, shared state между модулями и UI-утилиты.

---

## Архитектура

```
┌──────────────────────────────────────────────────┐
│  ModuleCustomCode.vue                            │
│  └── <ClientOnly>                                │
│      └── CustomCodeRuntime.vue                   │
│          ├── fetch .vue из codespace             │
│          ├── loadLibraries() — CDN через esm.sh  │
│          ├── vue3-sfc-loader → компиляция SFC    │
│          └── <component :is="..." :api :bindings>│
│                                                  │
│  usePortalDataLayer.js  (singleton)              │
│    ├── collections cache + dedup (fetchOnce)     │
│    ├── shared state (reactive)                   │
│    ├── pub/sub events                            │
│    └── user cache                                │
│                                                  │
│  useCustomCodeApi.js                             │
│    └── API-фасад → передаётся через prop `api`  │
│                                                  │
│  libraryRegistry.js                              │
│    └── каталог CDN-библиотек (esm.sh / unpkg)    │
│                                                  │
│  templateLibrary.js                              │
│    └── готовые шаблоны компонентов               │
└──────────────────────────────────────────────────┘
```

### Файлы

| Файл | Назначение |
|------|-----------|
| `components/modules/ModuleCustomCode.vue` | `<ClientOnly>` обёртка, рендерит `CustomCodeRuntime` |
| `components/custom-code/CustomCodeRuntime.vue` | Загрузка SFC из codespace, компиляция, error boundary |
| `composables/usePortalDataLayer.js` | Singleton реактивный data layer (кеш, shared state, pub/sub) |
| `composables/useCustomCodeApi.js` | API-фасад, передаётся через prop `api` |
| `components/custom-code/libraryRegistry.js` | Каталог CDN-библиотек |
| `components/custom-code/templateLibrary.js` | Готовые шаблоны компонентов |

---

## Props компонента

Каждый custom_code компонент получает три props:

| Prop | Тип | Описание |
|------|-----|----------|
| `api` | Object | API-фасад для доступа к данным, shared state, toast |
| `bindings` | Object | Привязки из конфига: `{ key: { type, id, raw } }` |
| `db` | String | Идентификатор воркспейса |

---

## API Reference

### Реактивные данные (кеш + дедупликация)

Все `use*` методы возвращают `ref` через singleton-кеш `fetchOnce`. Если два модуля на одной странице запрашивают одну коллекцию, запрос выполняется один раз.

| Метод | Возвращает | Rate limit | Описание |
|-------|-----------|------------|----------|
| `api.useCollection(typeId, params?)` | ref | 60/мин | Записи каталога |
| `api.useRecord(slug)` | ref | 60/мин | Одна запись по slug |
| `api.useReport(reportId, params?)` | ref | 60/мин | Данные отчёта |
| `api.useDocuments(params?)` | ref | 60/мин | Список документов |
| `api.useObjects(typeId, params?)` | ref | 60/мин | Записи произвольной таблицы (требует READ-грант) |
| `api.useDocument(docId)` | ref | 60/мин | Документ с блоками |
| `api.useTimeseries(sourceId, params?)` | ref | 30/мин | Данные таймсерий |
| `api.useRelated(objId, params?)` | ref | 30/мин | Граф-соседи записи |

### Write-операции

| Метод | Rate limit | Описание |
|-------|------------|----------|
| `api.createRecord(typeId, { name, fields, parentId })` | 30/мин | Создать запись (требует WRITE-грант) |
| `api.updateRecord(objectId, fields)` | 60/мин | Обновить запись |
| `api.runConnector(connectorId, params)` | 10/мин | Запустить коннектор (требует READ-грант) |
| `api.uploadFile(file, { fields, objectId })` | 20/мин | Загрузить файл. Возвращает `{ progress, result, error, abort, promise }`. Использует XHR для отслеживания прогресса. Backend: `POST /portal/api/upload`. |

### Навигация (sub-routing)

Для переходов между "под-страницами" внутри компонента (список → детали). Использует query params с префиксом `_cc_` — не конфликтует с host-роутером Nuxt.

| Метод | Описание |
|-------|----------|
| `api.navigate(view, params)` | Перейти: устанавливает `_cc_view` и доп. params в URL |
| `api.useRoute()` | Возвращает реактивный ref с текущими `_cc_*` params. Авто-cleanup при unmount. |

```vue
<script setup>
import { computed } from 'vue'
const props = defineProps({ api: Object })
const route = props.api.useRoute()          // { view: 'detail', id: '42' }
const view = computed(() => route.value.view || 'list')

function openDetail(id) {
  props.api.navigate('detail', { id })      // → ?_cc_view=detail&_cc_id=42
}
function goBack() {
  props.api.navigate('list')
}
</script>
```

> Для простых случаев (без deep-link) используйте обычный `ref('list')` — проще и надёжнее.

### Server Functions

| Метод | Rate limit | Описание |
|-------|------------|----------|
| `api.callFunction(name, args, { repo? })` | 60/мин | Вызвать серверную функцию из codespace-репозитория |

Выполняет `api/<name>.js` из указанного codespace-репозитория в изолированном V8-sandbox на сервере. По умолчанию `repo = 'portal-components'`.

```vue
<script setup>
const props = defineProps({ api: Object })

async function recalculate(itemId) {
  const result = await props.api.callFunction('calculate-price', { itemId })
  // result — данные, возвращённые серверной функцией
}

// С указанием другого репозитория:
const data = await props.api.callFunction('sync-data', { ids: [1, 2] }, { repo: 'my-tools' })
</script>
```

Подробная документация: [codespace-server-functions.md](../../docs/codespace-server-functions.md)

### AI

| Метод | Описание |
|-------|----------|
| `api.streamChat(message, { sessionId, sessionKey, pageHint, signal })` | Async generator токенов. Портальный AI-чат (`POST /portal/api/chat/message`). |
| `api.runAgent(message, { threadId?, agentSlug?, signal? })` | Async generator AG-UI событий. Оркестратор с доступом к swarm memory, agent-registry и behavioral patterns (`POST /portal/api/agent/run`). Требует `agent.enabled: true` в конфиге портала. |

**`streamChat`** — простой текстовый ответ без инструментов:

```vue
<script setup>
import { ref } from 'vue'
const props = defineProps({ api: Object })
const text = ref('')
const loading = ref(false)

async function send(message) {
  text.value = ''
  loading.value = true
  const ctrl = new AbortController()
  try {
    for await (const token of props.api.streamChat(message, { signal: ctrl.signal })) {
      text.value += token
    }
  } finally { loading.value = false }
}
</script>
```

**`runAgent`** — полноценный агент с инструментами, swarm memory и STATE_DELTA:

```vue
<script setup>
import { ref, reactive } from 'vue'
const props = defineProps({ api: Object })
const output = ref('')
const agentState = reactive({})
const currentTool = ref(null)

async function runFmea(partId) {
  output.value = ''
  const ctrl = new AbortController()
  for await (const event of props.api.runAgent(`Проведи FMEA для детали #${partId}`, {
    agentSlug: 'fmea-agent',
    signal: ctrl.signal,
  })) {
    if (event.type === 'TEXT_MESSAGE_CONTENT') output.value += event.content
    if (event.type === 'TOOL_CALL_START')      currentTool.value = event.toolName
    if (event.type === 'TOOL_CALL_END')        currentTool.value = null
    if (event.type === 'STATE_DELTA') {
      // JSON Patch RFC 6902 — обновляет /memories/{key} и /shared/{key}
      for (const op of event.delta) {
        if (op.op === 'add')    agentState[op.path] = op.value
        if (op.op === 'remove') delete agentState[op.path]
      }
    }
    if (event.type === 'RUN_FINISHED') break
  }
}
</script>
```

События `runAgent`:

| Событие | Поля | Описание |
|---------|------|----------|
| `RUN_STARTED` | — | Оркестратор запущен |
| `TEXT_MESSAGE_CONTENT` | `content: string` | Токен ответа |
| `TOOL_CALL_START` | `toolName: string, args: object` | Агент вызвал инструмент |
| `TOOL_CALL_END` | `toolName: string, result: any` | Инструмент вернул результат |
| `STATE_DELTA` | `delta: JSONPatch[]` | Изменения swarm memory (RFC 6902) |
| `RUN_FINISHED` | — | Оркестратор завершил работу |
| `error` | `message: string` | Ошибка |

### Polling (live-данные)

| Метод | Описание |
|-------|----------|
| `api.usePolling(fn, intervalMs?)` | Вызывает `fn` каждые N мс. Рекурсивный `setTimeout` (не `setInterval`). Возвращает `{ data, error, loading, pause, resume }`. Авто-пауза при скрытой вкладке. Экспоненциальный backoff с jitter при ошибках. Авто-cleanup при unmount. |

```vue
<script setup>
const props = defineProps({ api: Object, bindings: Object })
const { data, loading } = props.api.usePolling(async () => {
  const r = await fetch(`/api/v2/${props.db}/portal/api/tables/${props.bindings.source?.id}/objects`,
    { credentials: 'include' })
  return r.json().then(r => r.data)
}, 3000)
</script>
```

### Shared state

Все модули `custom_code` на одной странице разделяют один `PortalDataLayer` singleton — общий кеш, shared state и pub/sub. Используйте это для координации между несколькими компонентами.

| Метод | Описание |
|-------|----------|
| `api.shared(key, default?)` | Реактивный ref, общий для всех модулей на странице |
| `api.on(name, cb)` | Подписка на событие (возвращает unsubscribe) |
| `api.emit(name, data)` | Отправка события другим модулям |

**Пример: фильтр в одном модуле → список в другом**

Модуль A (фильтр):
```vue
<script setup>
const props = defineProps({ api: Object })
const category = props.api.shared('filter.category', '')
</script>
<template>
  <select v-model="category.value">
    <option value="">Все</option>
    <option value="electronics">Электроника</option>
  </select>
</template>
```

Модуль B (список, реагирует на фильтр):
```vue
<script setup>
import { computed } from 'vue'
const props = defineProps({ api: Object, bindings: Object })
const category = props.api.shared('filter.category', '')
const { data } = props.api.useCollection(props.bindings.source?.id)
const filtered = computed(() =>
  (data.value || []).filter(r => !category.value || r.category === category.value)
)
</script>
```

### Auth

| Метод | Описание |
|-------|----------|
| `api.getUser()` | Реактивный ref с записью клиента (EAV-поля) |
| `api.isAuthenticated` | Computed boolean |
| `api.refreshUser()` | Обновить данные клиента |

### UI

| Метод | Описание |
|-------|----------|
| `api.toast.success(msg)` | Зелёное уведомление |
| `api.toast.error(msg)` | Красное уведомление |
| `api.toast.info(msg)` | Синее уведомление |

Toast реализован через `CustomEvent('portal-toast', { detail: { severity, message } })` — перехватывается в `layouts/portal.vue`, отображается как всплывающее уведомление внизу справа (4 сек).

---

## Bindings

Привязки задаются в визуальном редакторе. Формат значения: `table:123`, `report:456`, `connector:7`.

При парсинге в `useCustomCodeApi` строка разбирается на объект и доступна через `props.api.bindings`:

```js
// Конфиг модуля
"bindings": { "products": "table:42", "report": "report:12" }

// В коде компонента — через props.api.bindings (НЕ props.bindings)
const typeId = props.api.bindings?.products?.id   // → 42
const reportId = props.api.bindings?.report?.id   // → 12

// Структура одной привязки:
// { type: 'table', id: 42, raw: 'table:42' }
```

> **Важно:** используйте `props.api.bindings.key.id`, а **не** `props.bindings.key` — `props.bindings` содержит сырые строки (`"table:42"`), парсинг происходит внутри API-фасада.

Привязка без `:` трактуется как `{ type: 'raw', id: value }`.

---

## Библиотеки

### Встроенные (всегда доступны)

- Vue 3 (`vue`)
- vue-chartjs (`vue-chartjs`)
- chart.js (`chart.js`)

### Опциональные (выбираются в редакторе)

Загружаются с CDN через `libraryRegistry.js`. CSS загружается автоматически.

| ID | Библиотека | Импорт | CSS | CDN |
|----|------------|--------|-----|-----|
| `primevue` | PrimeVue 4 | `primevue/button`, `primevue/datatable`, `primevue/column`, `primevue/inputtext`, `primevue/dialog`, `primevue/select`, `primevue/textarea`, `primevue/tag`, `primevue/progressbar` | Aura Light Blue тема | esm.sh |
| `leaflet` | Leaflet | через `window.L` (см. ниже) | leaflet.css | unpkg.com |
| `dayjs` | Day.js | `dayjs` | -- | esm.sh |
| `lodash-es` | Lodash | `lodash-es` | -- | esm.sh |
| `dompurify` | DOMPurify | `dompurify` | -- | esm.sh |
| `marked` | Marked | `marked` | -- | esm.sh |
| `sortablejs` | SortableJS | `sortablejs` | -- | esm.sh |
| `qrcode` | QRCode | `qrcode` | -- | esm.sh |
| `d3` | D3.js | `d3` | -- | esm.sh |
| `three` | Three.js | `three` | -- | esm.sh |
| `cytoscape` | Cytoscape.js | `cytoscape` | -- | esm.sh |
| `opencascade` | OpenCascade.js | `opencascade` | -- | esm.sh |

### Leaflet — особый случай

Leaflet **нельзя** загружать через `import` из esm.sh — Firefox блокирует cross-origin `dynamic import()` без заголовка `Cross-Origin-Resource-Policy` (esm.sh его не выставляет). Кроме того, leaflet — большой файл, и `fetch()` через `getFile()` может прерваться по Content-Length.

Правильный подход — загрузить leaflet через `<script>` тег из unpkg.com и использовать `window.L`:

```vue
<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
// НЕ импортируем leaflet — загружается через script тег ниже

const mapEl = ref(null)
let map = null

function loadScript(src) {
  return new Promise((resolve, reject) => {
    if (document.querySelector(`script[src="${src}"]`)) { resolve(); return }
    const s = document.createElement('script')
    s.src = src
    s.onload = resolve
    s.onerror = reject
    document.head.appendChild(s)
  })
}

onMounted(async () => {
  await loadScript('https://unpkg.com/leaflet@1.9.4/dist/leaflet.js')
  const L = window.L
  map = L.map(mapEl.value).setView([55.75, 37.62], 12)
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map)
})

onBeforeUnmount(() => { if (map) map.remove() })
</script>

<template>
  <div ref="mapEl" style="height:400px"></div>
</template>
```

CSS загружается автоматически через `libraryRegistry` (unpkg.com/leaflet@1.9.4/dist/leaflet.css) — достаточно добавить `leaflet` в `libraries` в конфиге модуля.

### Произвольные CDN-импорты

Любой bare-module import, не найденный в `moduleCache`, автоматически резолвится через `esm.sh`:

```js
import confetti from 'canvas-confetti'  // → import('https://esm.sh/canvas-confetti')
```

> **Firefox + esm.sh:** esm.sh не выставляет `Cross-Origin-Resource-Policy: cross-origin`, поэтому динамический `import()` может быть заблокирован в Firefox. Если библиотека не работает в Firefox — переходите на `<script>` тег с unpkg.com (как leaflet выше).

---

## CSS-переменные портала

Компоненты могут использовать CSS-переменные, заданные в branding-конфиге:

| Переменная | Назначение |
|-----------|------------|
| `--color-primary` | Основной цвет бренда |
| `--color-bg` | Фон страницы |
| `--color-surface` | Фон карточек |
| `--color-text` | Основной текст |
| `--color-text-secondary` | Вторичный текст |
| `--border-radius` | Скругление углов |
| `--card-border` | Граница карточек |
| `--card-shadow` | Тень карточек |
| `--font-heading` | Шрифт заголовков |
| `--font-body` | Шрифт текста |

---

## Конфиг модуля

```json
{
  "type": "custom_code",
  "config": {
    "repo": "portal-components",
    "file": "calculator.vue",
    "ref": "main",
    "libraries": ["primevue"],
    "bindings": {
      "products": "table:42",
      "report": "report:12"
    }
  }
}
```

| Поле | Тип | Описание |
|------|-----|----------|
| `repo` | String | Имя codespace-репозитория |
| `file` | String | Путь к `.vue` файлу в репозитории |
| `ref` | String | Ветка/тег/коммит (по умолчанию `main`) |
| `libraries` | String[] | ID опциональных библиотек из `libraryRegistry` |
| `bindings` | Object | Привязки `{ key: "type:id" }` |

---

## Шаблоны

Готовые шаблоны для быстрого старта (`templateLibrary.js`). Доступны в визуальном редакторе портала.

| ID | Название | Описание | Библиотеки | Нужен binding |
|----|----------|----------|------------|--------------|
| `data-table` | Таблица данных | Таблица с поиском и пагинацией | primevue | `source: "table:ID"` |
| `bar-chart` | Столбчатая диаграмма | График из данных отчёта | -- | `report: "report:ID"` |
| `contact-form` | Форма обратной связи | Форма с отправкой записи в таблицу | -- | `source: "table:ID"` (WRITE-грант) |
| `stats-cards` | Карточки статистики | KPI-метрики (кол-во, сумма, среднее) | -- | `source: "table:ID"` |
| `map-view` | Карта | Интерактивная карта с маркерами | leaflet | `source: "table:ID"` (поля `lat`, `lng`, `name`) |

---

## Жизненный цикл компонента

1. `ModuleCustomCode.vue` рендерит `<ClientOnly>` обёртку
2. `CustomCodeRuntime.vue` инициализирует `PortalDataLayer` singleton
3. Создаётся API-фасад через `createPortalApi(dataLayer, bindings)`
4. Загружается `.vue` файл из codespace (`GET /api/v2/:db/codespace/repos/:repo/blob/:ref/:file`)
5. Загружаются CDN-библиотеки из `libraryRegistry` (параллельно)
6. vue3-sfc-loader компилирует SFC, получая `moduleCache` со всеми библиотеками
7. Скомпилированный компонент рендерится с props: `api`, `bindings`, `db`
8. При ошибке — показывается error boundary, событие `@error` эмитится наверх
9. При размонтировании — `<style>` и `<link>` теги чистятся

---

## Изоляция стилей

`<style>` блоки пользовательского компонента компилируются в браузере и добавляются
в `document.head`, то есть по устройству они глобальные. Чтобы правило вида
`.card { … }` внутри компонента не перекрашивало оболочку портала,
`CustomCodeRuntime.vue` рендерит обёртку `<div class="custom-code-root">`
(`display: contents`, в раскладке не участвует) и оборачивает каждый вставляемый
блок стилей в `@scope (.custom-code-root) { … }`.

Что из этого следует для автора компонента:

- **Наружу стили не выходят.** Даже неscoped `<style>` действует только внутри компонента.
- **Наследование работает как раньше.** `@scope` ограничивает подбор селекторов, но не
  наследование: `--color-primary`, `--font-body`, `--border-radius` и прочие токены темы
  портала по-прежнему приходят от `.portal-layout`.
- **`@keyframes`, `@media`, `@supports` внутри `@scope` работают** — проверено в браузере.
- **`@import` и `@charset` не оборачиваются** — они допустимы только в начале таблицы стилей.

Поддержка `@scope` определяется на лету (пробная таблица стилей, а не список версий).
Там, где её нет — Firefox ESR 140 и 115 — стили вставляются как раньше, без изоляции:
хуже, чем было, не становится, но и лучше тоже.

Внутрь компонента правила оболочки не достают: в `assets/portal.css` не осталось
селекторов без области видимости, а односложные имена вроде `.dot` переименованы.
Исключение сделано намеренно: `.portal-layout h1, h2, h3 { font-family: … }` —
заголовочный шрифт из настроек портала применяется и к заголовкам внутри компонента.

---

## Ограничения

- Только `<script setup>` (Options API не поддерживается vue3-sfc-loader в этом режиме)
- Multi-file компоненты поддерживаются: относительные импорты (`./Foo.vue`) резолвятся через codespace API. `CustomCodeRuntime.vue` использует `getFile` callback vue3-sfc-loader для загрузки зависимых `.vue` файлов из того же codespace-репозитория
- SSR не поддерживается (модуль рендерится только на клиенте через `<ClientOnly>`)
- TypeScript поддерживается (`<script setup lang="ts">`)
- Все модули на странице разделяют один `PortalDataLayer` singleton (один кеш, один shared state)

## Безопасность

### Content Security Policy

Портал устанавливает CSP через `portal/server/middleware/csp.ts`. Ключевые директивы для custom_code:

| Директива | Значение | Причина |
|-----------|---------|---------|
| `script-src` | `'self' 'unsafe-inline' 'unsafe-eval' https://esm.sh https://unpkg.com` | `'unsafe-eval'` — vue3-sfc-loader (`new Function()`); `'unsafe-inline'` — обязательно для Nuxt 3 SSR-гидрации (inline-скрипты `__NUXT_DATA__`); `https://unpkg.com` — leaflet через `<script>` тег |
| `style-src` | `'self' 'unsafe-inline' https://fonts.googleapis.com https://esm.sh https://unpkg.com` | `'unsafe-inline'` — `addStyle()` vue3-sfc-loader и `:style` биндинги Vue; `https://unpkg.com` — leaflet.css |
| `connect-src` | `'self' https://esm.sh https://*.tile.openstreetmap.org` | API портала, fetch к esm.sh, тайлы Leaflet |
| `frame-ancestors` | `'none'` | Защита от clickjacking |

> **Важно:** `'unsafe-inline'` в `script-src` **обязателен** для Nuxt 3 SSR. Без него гидрация падает с ошибкой `TypeError: 'target' argument of Proxy must be an object, got undefined` — браузер блокирует инлайн-скрипты `__NUXT_DATA__`, которые Nuxt вставляет на странице для передачи состояния с сервера на клиент.

### Изоляция кода

Кастомные компоненты выполняются в контексте основного приложения портала (не в `<iframe sandbox>`). Это сознательный выбор — код пишут только администраторы через codespace, не пользователи портала. При необходимости более строгой изоляции (публичный редактор кода) требуется переход на iframe-sandbox с `postMessage` API.

### esm.sh и цепочка поставок

Опциональные библиотеки загружаются с `esm.sh` CDN. Версии пинятся (`primevue@4`, `leaflet@1`), но без SRI-хешей. Для критичных деплойментов рекомендуется самохостинг библиотек или использование jsDelivr с SRI.

---

## Отладка

### Ошибки компиляции и рантайма

Когда компонент бросает ошибку (во время компиляции или выполнения), `CustomCodeRuntime` перехватывает её через `onErrorCaptured` и показывает в месте компонента:

```
Ошибка в компоненте: <текст ошибки>
```

Полный стектрейс виден в консоли браузера (DevTools → Console). `console.log` внутри компонента работает в обычном режиме.

### Частые ошибки

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `File not found: <file>` | Файл не существует на ветке `ref` | Проверьте что файл закоммичен на нужную ветку |
| `403 FILE_NOT_IN_CONFIG` | Файл не указан в конфиге модуля | Сохраните конфиг портала с корректными `repo`/`file` |
| `Relative imports not supported: ./foo` | Попытка импортировать соседний файл | Вынесите код в один файл или используйте CDN |
| `Options API не поддерживается` | `export default { ... }` вместо `<script setup>` | Перепишите на `<script setup>` |
| Компонент не перезагружается после коммита | Браузер кешировал ответ | Обновите страницу (Ctrl+Shift+R) |
| `TypeError: 'target' argument of Proxy must be an object, got undefined` | CSP блокирует инлайн-скрипты Nuxt SSR (`'unsafe-inline'` отсутствует в `script-src`) | Добавьте `'unsafe-inline'` в `script-src` в `csp.ts` |
| `_dayjs.default is not a function` | esm.sh-модуль вернул ES namespace объект | Используйте `.default` или деструктурируйте: `const dayjs = (await import(...)).default` |
| `Cross-Origin Request Blocked` (leaflet в Firefox) | Firefox блокирует `dynamic import()` без `Cross-Origin-Resource-Policy` | Используйте `<script>` тег вместо `import()` (см. раздел Leaflet выше) |
| `⚠ Привязка X не задана` | Компонент читает `props.bindings.X` (сырая строка) вместо `props.api.bindings.X.id` | Используйте `props.api.bindings?.X?.id` |

### Рабочий процесс разработки

1. Создайте `.vue` файл локально и разработайте компонент
2. Закоммитьте в codespace-репозиторий (через `git push` или MCP-инструмент `commit_portal_component`)
3. Откройте портал и обновите страницу — компонент перекомпилируется автоматически
4. Ошибки смотрите в DevTools → Console (тег `[custom_code]`)

> Компонент компилируется при каждой загрузке страницы (нет отдельного build-шага). Изменение файла в репо сразу доступно после обновления страницы.
