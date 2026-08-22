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

Подробная документация: [codespace-server-functions.md](../../docs/guides/codespace-server-functions.md)

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
| `RUN_STARTED` | `convId` | Оркестратор запущен |
| `TEXT_MESSAGE_CONTENT` | `content: string` | Кусок печатаемого текста. **Это ещё не ответ**: до `thinking_boundary` здесь идёт ход работы агента |
| `thinking` | `content: string` | Рассуждение агента отдельным куском (модели с extended thinking) |
| `thinking_boundary` | — | Всё, что напечатано ДО этой отметки, было ходом работы, а не ответом. Получив её, обнулите накопленный текст (сохранив отдельно, если показываете ход работы) |
| `TOOL_CALL_START` | `toolName: string, args: object` | Агент вызвал инструмент |
| `TOOL_CALL_END` | `toolName: string, result: any` | Инструмент вернул результат |
| `warning` | `content: string` | Предупреждение о самом разговоре: подменён собеседник, назван несуществующий инструмент |
| `truncated` | — | Ответ обрезан ограничением длины |
| `elicit` | `prompt, schema, toolCallId, agentId` | Агент задал уточняющий вопрос |
| `hitl` | `payload` | Агент ждёт подтверждения действия |
| `STATE_DELTA` | `delta: JSONPatch[]` | Изменения swarm memory (RFC 6902) |
| `RUN_ERROR` | `outcome: string, message: string` | Прогон дошёл до конца и не дал итога |
| `RUN_FINISHED` | `convId, outcome: string, content: string` | Завершение. `content` — **итоговый ответ целиком, без хода работы**; `outcome` — одно из `answered`, `empty`, `exhausted`, `aborted`, `hitl` |
| `error` | `message: string` | Отказ маршрута — в отличие от `RUN_ERROR`, случается и до запуска агента |

**Как читать поток правильно.** Текст ответа берите из `RUN_FINISHED.content`.
Куски `TEXT_MESSAGE_CONTENT` нужны только для живой печати; если собираете итог
из них, обнуляйте накопленное на `thinking_boundary` — иначе в отчёт попадёт
«Сейчас соберу данные…». Успех определяйте по `outcome === 'answered'`, а не по
непустоте текста: болтовня непуста всегда. Старый сервер полей `outcome` и
`content` у `RUN_FINISHED` не шлёт — держите запасной путь на накоплении.

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

Резолвятся из `node_modules` самого портала — ни одного запроса к CDN.

| Импорт | Что это |
|--------|---------|
| `vue` | Vue 3. Та же копия, что у приложения (`@vue/runtime-core`, `@vue/runtime-dom`, `@vue/reactivity`, `@vue/shared` указывают на неё же) |
| `vue-chartjs`, `chart.js` | Графики |
| `marked`, `dompurify` | Markdown и санитизация. `dompurify` резолвится в `isomorphic-dompurify` |
| `primevue` | PrimeVue 4, **только через бочку**: `import { Button, Tag, InputText } from 'primevue'`. Импортируется статически, чтобы `PrimeVueSymbol` совпал с зарегистрированным в приложении плагином — иначе `inject()` внутри компонентов PrimeVue не находит конфигурацию. Путь вида `primevue/button` в `moduleCache` не лежит и уедет на esm.sh отдельной копией |

Выбирать `primevue` в списке библиотек не нужно — в `libraryRegistry` его нет.

### Опциональные (выбираются в редакторе)

Загружаются с CDN через `libraryRegistry.js`. CSS загружается автоматически.

| ID | Библиотека | Импорт | CSS | CDN |
|----|------------|--------|-----|-----|
| `leaflet` | Leaflet | через `window.L` (см. ниже) | leaflet.css | unpkg.com |
| `dayjs` | Day.js | `dayjs` | -- | esm.sh |
| `lodash-es` | Lodash | `lodash-es` | -- | esm.sh |
| `dompurify` | DOMPurify | `dompurify` | -- | esm.sh |
| `marked` | Marked | `marked` | -- | esm.sh |
| `sortablejs` | SortableJS | `sortablejs` | -- | esm.sh |
| `qrcode` | QRCode | `qrcode` | -- | esm.sh |
| `d3` | D3.js | `d3` | -- | esm.sh |
| `three` | Three.js | `three` | -- | esm.sh |
| `cytoscape` | Cytoscape.js | `cytoscape` (dagre-layout уже зарегистрирован через `cytoscape.use`) | -- | esm.sh |
| `opencascade` | OpenCascade.js | `opencascade` | -- | esm.sh |

> `marked` и `dompurify` есть и во встроенных, и здесь. Выбор их в списке библиотек **заменяет** копию из `node_modules` на копию с CDN: `moduleCache` дополняется опциональными библиотеками после встроенных.

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

## Библиотека `@kit`

Ядро для порталов: чтение данных, разбор значений EAV, показ дат и сумм, три
состояния отсутствия. Исходники — `portal-kit/` в этом репозитории, полное
описание выпуска и версий — `portal-kit/README.md`.

### Зачем она заведена

У портала нет сборщика, репозиторий у каждого проекта свой, общего нет —
поэтому слой чтения копировался из проекта в проект, и копии разошлись молча:

- `numVal` объявлен в четырёх местах и на одних и тех же входах отвечает
  по-разному: в фонде пустая ячейка даёт `0` (отсутствие попадает в суммы как
  измеренный ноль), в СППР — `null`; `"3,5"` в фонде тоже `0`;
- `stripRef` — девять объявлений под одним именем, возвращающих разное: где
  идентификатор (`5`), где имя (`"Проект"`). Перенос кода между проектами
  подменяет одно другим.

Такие расхождения не ломают экран заметно — они меняют числа и подписи.
`@kit` — одна копия, выложенная отдельно от проектов и закреплённая версией.

### Что в ней есть

Восемь заготовок. Полное описание пропсов и слотов — в `portal-kit/README.md`
и у агента через `kit_get_component`; здесь коротко, чтобы было видно, что
брать.

| Заготовка | Для чего |
|---|---|
| `AiPanel` | боковой ИИ-чат: вопрос уходит на `POST /agent/run`, ответ приходит потоком SSE; беседы, карточки вызванных инструментов, ход работы агента, подтверждение отложенного действия. Ответ агента печатается **разметкой** (заголовки, списки, жирный, курсив, код, ссылки со схемой `http`/`https`/`mailto`): узлы строятся `h()`, строка HTML не собирается ни разу, `v-html` в библиотеке нет — вставлять нечего, поэтому и очищать нечего. Выключается пропом `markdown: false`. Разбор вывезен отдельно как `parseMarkdown` |
| `TeamchatPanel` | командный чат: комнаты, темы, лента с ответами, правкой, реакциями, вложениями, упоминаниями, закреплениями, звёздами и поиском |
| `KbBrowser` | мета-КБ: перечень тем и разборы |
| `DataTable` | таблица: сортировка, счётчик полноты, слоты ячеек; пустое значение и настоящий ноль показываются по-разному |
| `Source` | чтение без вёрстки — отдаёт строки, полноту и отказ в слот, разметка ваша |
| `CitationText` | текст со ссылками на источники; разметка из базы не вставляется как HTML |
| `StateChip` | плашка состояния: слово, форма, цвет — именно в этом порядке |
| `EmptyState` | различает «пусто», «прочитать не удалось» и «неприменимо» |

**Два чата — не дубль.** `AiPanel` — разговор с ИИ-агентом, тот самый, что в
платформе открывается кнопкой сбоку. `TeamchatPanel` — командный чат с
комнатами и темами. Имена выбраны так, чтобы это было видно без чтения кода.

Читают заготовки только ядром (`readAll`, `fetchOne`) — то есть получают
доказанную полноту и внятный отказ даром. Пишут своими вызовами: ядро шлёт
только `GET`.

Оговорка, без которой предыдущая фраза вводит в заблуждение (найдено внешней
проверкой 12.08.2026): «ядро не пишет» — это про библиотеку, а не про
последствия. У источника `chat-topics` в реестре стоит пометка, что **чтение тем
вдобавок ПИШЕТ членство в комнате**: пишет маршрут, а не мы, но со стороны
человека список тем, открытый на просмотр, записывает его в комнату. Читая
источник, смотрите его пометку в `src/core/sources.js` — «только GET» не значит
«без последствий».

### Свой агент у портала

По умолчанию `AiPanel` разговаривает с оркестратором платформы — тем же, что в
рабочем месте. Чтобы отвечал агент вашей темы:

1. завести строку в реестре агентов воркспейса: `type: internal`, своя
   подсказка, перечень инструментов, модель;
2. вписать её слаг в `agent.allowedSlugs` конфига портала — иначе сервер слаг
   отбросит и ответит оркестратор;
3. передать панели `:agent-slug="…"`.

Набор инструментов у такого агента ровно тот, что назван в строке. Имя, которого
нет ни среди платформенных, ни среди инструментов воркспейса, будет названо
предупреждением в потоке ответа (панель печатает его шагом рядом с ответом с
версии `0.9.2`), а не проглочено.

Чего делать НЕ надо: вписывать в `allowedSlugs` платформенный `portal-agent`.
Это агент администратора портала — он читает заказы и корзины всех клиентов,
тикеты, выручку, профиль по email и конфиг целиком, и все эти инструменты
портальным гейтом не закрыты.

Почему так устроено и чего это стоит — `docs/adr/024-workspace-defined-agents.md`.

### Как подключить

Версия задаётся **в конфиге модуля**, а не в коде компонента:

```json
{ "type": "custom_code", "config": { "repo": "fund-app", "file": "app.vue", "ref": "main", "kit": "0.6.0" } }
```

В компоненте:

```js
import { numVal, readAll, fmtDate, NO_DATA } from '@kit'
```

Спецификатор `@kit` перехватывается в `getFile` **до всех прочих веток**
(иначе голое имя уехало бы на esm.sh) и скачивается с
`/assets/kit/<версия>/kit.js`. Артефакт собран с `external: ['vue']`, поэтому
отдаётся загрузчику как `{ getContentData, type: '.mjs' }`, а не кладётся в
`moduleCache` готовым модулем: голое `from "vue"` внутри браузер без import map
не резолвит, а vue3-sfc-loader разрешает его сам — в ту же копию Vue, что у
приложения.

Если поле `kit` в конфиге пусто, импорт отвечает отказом дословно:

```
Библиотека '@kit' не подключена: в конфиге модуля не задана версия
```

Если версия задана, но не выложена — `Библиотека '@kit' версии <N> не найдена: 404`.
Молчаливого ухода на CDN нет ни в том, ни в другом случае.

### Что внутри

| Имя | Что делает | Чем отличается от наивной реализации |
|-----|-----------|--------------------------------------|
| `KIT_VERSION` | Версия артефакта строкой | Совпадение с `package.json` проверяется тестом сборки |
| `numVal(v)` | Число из значения EAV | Отсутствие — всегда `null`, **никогда `0`**. Локаль русская. Ссылка (`"Имя (id:5)"`, `"5:Имя"`) даёт идентификатор, а `"12:30"` не читается как ссылка на объект 12 |
| `refText(v)` | Имя из ссылочного значения | Понимает обе формы записи и объект; `null` даёт `''`, а не строку «null» |
| `refId(v)` | Идентификатор из ссылочного значения либо `null` | Разведён с `refText` намеренно — это и есть тот самый `stripRef` |
| `field(row, name)` | Поле записи по **имени** колонки | Портальный маршрут адресует поля именами, не идентификаторами реквизитов. Читает только собственные свойства: `'constructor'` не достанет метод с прототипа. Значение отдаётся сырым |
| `NO_DATA`, `NO_VALUE`, `NOT_APPLICABLE` | Три состояния отсутствия, знаки `—`, `·`, `×` | Общий прочерк на всех трёх врёт в двух случаях из трёх |
| `absenceOf(value, opts)` | Одно из трёх состояний либо `null`, если значение заполнено | `opts`: `applicable`, `normKnown`. `NaN` и бесконечность — `NO_DATA`. Пустой список и пустой объект отсутствием **не** считаются |
| `isAbsent(v)` | Одно ли из трёх состояний перед нами | Опознаёт по тождеству: копия через развёртывание (`{...NO_DATA}`) состоянием не является |
| `isCancelled(e)` | Отмена ли это | Обходит цепочку `cause`, поэтому завёрнутая отмена не читается как отказ сервера |
| `cancelledError(what, why)` | Своя отмена с указанием, что и почему отменено | В журнале остаётся не голое «отменено» |
| `parseSource(source)` | Разбор `"table:482"` или `{type, id}` в `{type, id, raw}` | `raw` собирается заново: `"table:0482"` и `"table:482"` дают один ключ. `"table:"` и `"table:1.5"` — отказ, а не источник номер ноль. **Явный ноль тоже отказ** там, где идентификатор — номер записи: записи №0 не бывает, маршрут таблиц отвечает на неё 400 (с 0.9.11). Ноль остаётся обязательным у маршрутов на весь воркспейс — `catalog:0` |
| `SOURCE_TYPES` | Восемнадцать родов. Данные воркспейса: `table`, `doc`, `docs`, `report`, `timeseries`, `record`, `related`, `catalog`, `decisions`. Мета-КБ: `kb-topics`, `kb-debates`. Командный чат: `chat-topics`, `chat-messages`, `chat-pinned`, `chat-starred`, `chat-recent`. ИИ-чат: `agent-conversations`, `agent-messages` | Перечень был назван шестью родами — двенадцати не хватало (внешняя проверка 12.08.2026). У каждого рода в `src/core/sources.js` лежит пометка о его особенностях, и она подробнее этой строки: например, `chat-starred` **нельзя листать** — порядок задан временем постановки звезды, а курсор отбирает по номеру сообщения, поэтому во вторую страницу не попадёт отмеченное раньше, но с бо́льшим номером |
| `fetchPage(source, opts)` | Одна страница → `{items, total, totalKnown, offset, limit}` | `totalKnown` отделяет обещание сервера от нашего счёта прочитанного. Ответ без списка строк — отказ, а не пустая таблица |
| `readAll(source, opts)` | Обход целиком → `{items, total, pages, complete, reason}` | `complete` доказывается неполной страницей, а не арифметикой по `total` |
| `fmtDate(v)` | Дата как `ДД.ММ.ГГГГ` | Разбирает обе формы хранения — строку `ГГГГ-ММ-ДД` и эпоху (10 и 13 знаков), в UTC. Неразобранное показывает знаком отсутствия, а не догадкой |
| `money(v, {unit})` | Сумма с неразрывными разделителями разрядов и знаком валюты | Отсутствие — знак `NO_DATA`; `NaN` и `∞` тоже, а не «не число» и «∞» на экране. **Ссылка суммой не становится**: `«Проект (id:5)»` даёт знак отсутствия, а не «5 ₽» — номер записи никогда не является суммой (с 0.9.11). Знаков после запятой ноль или два, третьего не бывает |
| `pct(v, {digits})` | Доля в процентах с постоянным числом знаков | Постоянное число знаков: доля то с одним знаком, то без читается как разная точность замера. **Ссылка долей не становится** — то же правило, что у `money` (с 0.9.11) |

Общие для `fetchPage` и `readAll` `opts`: `db` (обязательно), `limit`
(по умолчанию 200), `offset`, `parentId`, `signal`, `fetchFn`; у `readAll`
дополнительно `maxPages` (по умолчанию 50). Запрос уходит с
`credentials: 'include'`.

### Каталог: как состав библиотеки узнаёт агент

Таблица выше набрана руками и живёт в документе — она объясняет, а не
удостоверяет. Машиночитаемый состав лежит рядом с самим артефактом:
`/assets/kit/<версия>/manifest.json`. Он **собирается из исходников при сборке**
библиотеки (`portal-kit/scripts/build-manifest.mjs`), поэтому отстать от кода не
может: перечень, набранный руками, разошёлся бы с кодом на первом же изменении
пропса и никого бы об этом не предупредил. Тест сборки требует, чтобы манифест
перечислял ровно то, что вывозит точка входа, — ни больше, ни меньше.

Агенту манифест доступен тремя инструментами платформы (группа `portal`,
все три — только чтение):

| Инструмент | Что отдаёт |
|---|---|
| `kit_list_components(version?, kind?, search?)` | Перечень: имя, род, однострочное описание, модуль. `kind` — `function`, `constant` или `component` |
| `kit_get_component(name, version?)` | Одну запись подробно; для компонентов — пропсы и слоты, читаемые токены и устойчивые имена классов |
| `kit_get_tokens(version?, kind?, component?)` | Словарь оформительских токенов и устойчивые классы всех заготовок. См. следующий раздел |
| `kit_list_versions()` | Какие версии выложены и какая последняя. Порядок считается числами, поэтому `0.1.10` старше `0.1.2` |

Без `version` отвечает последняя выложенная версия. Свою версию проект берёт из
поля `kit` в конфиге модуля — она задана там, а не в коде компонента.

Тот же каталог читается и без агента: `GET /:db/portal/api/kit/catalog`
с необязательными `version`, `kind`, `search` — порталу он нужен для подсказок
в редакторе.

Отказы различают четыре разных положения дел, и **пустой перечень ни одним из
них не является** — пустой перечень означает только пустой отбор:

| Код | Что произошло | Код HTTP |
|---|---|---|
| `KIT_NOT_DEPLOYED` | Библиотеки на сервере нет вовсе либо не выложено ни одной версии | 503 |
| `KIT_VERSION_NOT_FOUND` | Такая версия не выложена; в сообщении перечислены выложенные | 404 |
| `KIT_CATALOG_MISSING` | Версия выложена, а каталога у неё нет — так у ранних версий, выложенных до того, как манифест стал ехать вместе с артефактом | 404 |
| `KIT_CATALOG_BROKEN` | Манифест есть, но прочитать его нельзя либо он объявляет себя версией, отличной от той, под которой лежит | 500 |

### Свои стили и анимации для заготовок

Своих цветов, размеров и длительностей у заготовок нет — всё идёт через
объявленный словарь токенов. **Имена берутся из словаря, а не из головы:**
имени, которого в нём нет, не существует, `var(--kit-color-txt)` разрешится в
ничто, и раздел выйдет структурно верным и полностью без оформления. Поведение
при этом не сломается, и тесты промолчат — увидеть можно только на отрисованной
странице.

Откуда взять имена:

- агенту — `kit_get_tokens()`; поле `usage` каждого токена уже содержит готовую
  строку, писать надо её целиком, вместе с запасным значением;
- человеку — таблица в `portal-kit/README.md`, раздел «Свои стили и анимации»;
- из кода — `import { KIT_TOKENS } from '@kit'`.

Второго входа для словаря пока нет: `GET /:db/portal/api/kit/catalog` отдаёт
только перечень заготовок, токенов в его ответе не будет. Это пробел, а не
решение, — сказано, чтобы отсутствие поля не читалось как «словарь пуст».

Четыре уровня свободы, по возрастанию — берите первый, которого хватает:

1. **Переопределить токен.** Меняет вид всех заготовок разом, о внутренностях
   знать не надо:

   ```css
   .my-section { --kit-color-accent: #7c3aed; --kit-radius: 12px; }
   ```

2. **Передать свои классы пропом `ui`** — точечно, на именованный узел, прямо
   из разметки:

   ```vue
   <DataTable :rows="rows" :columns="cols" :ui="{ row: 'my-row', head: 'my-head' }" />
   ```

   Классы **дописываются** к нашим, не заменяя их. Перечень ключей у каждой
   заготовки свой (`root`, `row`, `cell`, `compose`, `send`…) и берётся полем
   `uiKeys` из `kit_get_component`; таблица для человека — в
   `portal-kit/README.md`. Ключи — часть договора, их переименование ломающее;
   ключа, которого в перечне нет, не существует — заготовка его не примет и
   скажет об этом в консоли. У `Source` поля `uiKeys` нет вовсе: он headless,
   своих узлов не имеет.

3. **Зацепиться за устойчивый класс.** У каждой заготовки объявлен корневой
   класс (`kit-ai`, `kit-chat`, `kit-kb`, `kit-tbl`, `kit-cite`, `kit-chip`,
   `kit-empty`) и имена на ключевых внутренних узлах; полный перечень — поле
   `classes` в ответе `kit_get_tokens`. Они часть договора, их переименование
   ломающее. Всё прочее внутри закрыто `scoped` и может измениться.

4. **Отдать свою вёрстку слотом** — когда нужен не другой вид, а другая
   разметка. Слоты перечислены в манифесте у каждого компонента.

Токен ищется цепочкой «наш → токен оболочки портала → запасное значение»:
`var(--kit-color-text, var(--color-text, #1f2328))`. Средним звеном стоят
настоящие имена портала (`--color-text`, `--color-surface`, `--color-border`,
`--color-primary`, `--border-radius` и ещё пять), поэтому портал со своей темой
окрашивает заготовки, ничего про библиотеку не зная. Запасные значения оставлены
сознательно — заготовка попадает и в оболочку без темы, где токен без запасного
разрешился бы в ничто; выдуманные имена ловятся не запасными, а проверкой при
сборке библиотеки. Обоснование целиком — в шапке `portal-kit/src/core/tokens.js`.

Движение стоит там, где несёт смысл: появление сообщения в ленте, появление
строки таблицы, появление плашки на месте содержимого. Движутся только
`transform` и `opacity`. `prefers-reduced-motion` обработан у всех заготовок с
вёрсткой, и уменьшенное движение **не равно** отсутствию движения: сдвиг
обнуляется, длительности укорачиваются, появление остаётся видимым. Сделано
переопределением токена, поэтому ослабление достаётся и вашим собственным
анимациям, если вы построили их на `--kit-motion-*`.

Отказы словаря, помимо четырёх общих для каталога:

| Код | Что произошло |
|---|---|
| `KIT_TOKENS_MISSING` | Каталог версии есть, а словаря в нём нет — так у версий по `0.4.0` включительно, они собраны до появления раздела |
| `KIT_NO_STYLING` | У названной заготовки нет вёрстки (`Source`): она отдаёт данные в слот и не читает ни одного токена. Оформлять в ней нечего — размечайте слот сами |

Оба отказа заведены по той же причине, что и четыре предыдущих: пустой словарь
читался бы как «оформлением управлять нечем», а это неправда.

### Что легко понять неправильно

- **`numVal` не отдаёт `0` при отсутствии.** `numVal('')`, `numVal(null)`,
  `numVal('   ')` — это `null`. Ноль возвращается только там, где ноль записан:
  `numVal('0') === 0`. Проверяйте `=== null`, а не истинность.
- **Запятая — десятичный разделитель.** `numVal('1,000')` это **единица**, а
  `numVal('1 000')` — тысяча. Английский формат намеренно не поддержан:
  отличить его от русского без знания источника нельзя, а угадывание портило бы
  числа молча.
- **`refText` и `refId` — разные функции.** Одна отдаёт `"Проект"`, другая `5`.
  Не подставляйте одну вместо другой «по смыслу имени» — именно на этом
  разошлись девять копий `stripRef`.
- **`readAll` возвращает `complete`, и его надо проверять.** Полнота
  доказывается неполной страницей, а не сверкой с `total`: сервер обещает и
  больше, и меньше, чем отдаёт. `complete: false` бывает при исчерпанном
  потолке страниц, при недоборе против обещанного и при пересечении страниц по
  `id`; причина названа словами в `reason`. `reason` может быть непустым и при
  `complete: true` — например, когда сервер отдал больше, чем обещал.
- **Дочерняя таблица требует `parentId`.** Портальный маршрут всегда фильтрует
  по родителю, и без `parentId` дочерняя таблица отдаёт ноль записей — что
  читается как «детей нет». Ноль здесь законный номер, поэтому передаётся и он.
- **Три состояния отсутствия различаются.** `NO_DATA` — нечем измерить,
  `NO_VALUE` — нормы или правила нет, `NOT_APPLICABLE` — вопрос не имеет смысла
  для этого объекта. Состояния опознаются по тождеству и переживают только те
  переносы, где объект остаётся тем же: после `JSON.parse(JSON.stringify(…))`
  это уже отпечаток, восстанавливать по полю `kind`.
- **Пустой список — не отсутствие.** `absenceOf([])` и `absenceOf({})` дают
  `null`: это состоявшееся измерение с итогом «ничего нет». Разница та же, что
  между «таблица пуста» и «таблицы нет».

### Пример: чтение таблицы с честным показом неполноты

```vue
<script setup>
import { ref, onMounted } from 'vue'
import { readAll, field, money, NO_DATA } from '@kit'

const props = defineProps({ api: Object, db: String })
const rows = ref([])
const note = ref('')

onMounted(async () => {
  const r = await readAll(props.api.bindings.source, { db: props.db })
  rows.value = r.items
  note.value = r.complete ? '' : `прочитано не всё: ${r.reason}`
})
</script>

<template>
  <p v-if="note">{{ note }}</p>
  <ul>
    <li v-for="row in rows" :key="row.id">
      {{ field(row, 'Наименование') || NO_DATA.mark }} — {{ money(field(row, 'Сумма')) }}
    </li>
  </ul>
</template>
```

`props.api.bindings.source` — это `{type, id, raw}`, и `parseSource` принимает
его как есть. Привязки нет — `readAll` отказывает громко («источник не задан»),
а не читает пустоту.

### Версионирование

Версия закреплена в конфиге проекта, поэтому правка библиотеки не трогает уже
работающие порталы: пока в конфиге стоит прежний номер, портал грузит именно его.
Обновление — правка одного поля; переводить порталы надо по одному. Выложенная
версия неизменяема (nginx раздаёт `/assets/kit/<версия>/kit.js` с
`Cache-Control: immutable`), старые версии с прода не убираются — на них
ссылаются работающие порталы.

Ломающим изменением — требующим новой версии и осознанного перевода каждого
портала — считается: убранный или переименованный экспорт; изменение состава
возвращаемого объекта (`readAll`, `fetchPage`, `parseSource`); изменение ответа
функции на прежнем входе (`numVal` вернул `0` вместо `null`, `fmtDate` сменил
формат, знак состояния отсутствия стал другим); новый обязательный параметр или
обязательное поле в `opts`; превращение мягкого отказа в исключение и наоборот.
Знаки `—`, `·`, `×` и формат даты — часть договора, а не оформление.

Ломающим **не** является: новый экспорт, новое необязательное поле в `opts`,
новое поле в возвращаемом объекте, уточнение текста сообщения об ошибке.

### Как проверить, что подключилось

1. Напечатайте `KIT_VERSION` — это единственный способ узнать, какая версия
   реально загружена, а не какая записана в конфиге:
   `import { KIT_VERSION } from '@kit'`.
2. Вкладка Network, запрос `/assets/kit/<версия>/kit.js`. Ответ 404 означает,
   что версии нет на проде; отсутствие запроса вовсе — что до импорта дело не
   дошло (компонент упал раньше).
3. Отказ на месте компонента читается по тексту:
   - `Библиотека '@kit' не подключена…` — поле `kit` в конфиге пусто;
   - `…версии X не найдена: 404` — версия не выложена;
   - `Failed to fetch <имя> from all CDNs` — спецификатор до ветки `@kit` не
     дошёл и ушёл на CDN как обычный пакет. Перехват стоит первым, так что это
     признак опечатки в имени: `@Kit`, `kit`, `@kit '` и подобное.

Запрос идёт с `cache: 'force-cache'`, но переход на новую версию меняет адрес,
поэтому устаревшего тела библиотеки из кэша браузера не бывает.

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
    "kit": "0.6.0",
    "libraries": ["dayjs"],
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
| `kit` | String | Версия библиотеки `@kit`. Пусто — импорт `@kit` отказывает (см. ниже) |
| `libraries` | String[] | ID опциональных библиотек из `libraryRegistry` |
| `bindings` | Object | Привязки `{ key: "type:id" }` |

---

## Жизненный цикл компонента

1. `ModuleCustomCode.vue` рендерит `<ClientOnly>` обёртку
2. `CustomCodeRuntime.vue` инициализирует `PortalDataLayer` singleton
3. Создаётся API-фасад через `createPortalApi(dataLayer, bindings)`
4. Загружается `.vue` файл из codespace (`GET /api/v2/:db/portal/api/codespace/:repo/blob/:ref/:file`)
5. Загружаются CDN-библиотеки из `libraryRegistry` (последовательно, по списку `libraries`)
6. vue3-sfc-loader компилирует SFC, получая `moduleCache` со всеми библиотеками; `@kit` и относительные импорты доходят до `getFile`
7. Скомпилированный компонент рендерится с props: `api`, `bindings`, `db`
8. При ошибке — показывается error boundary, событие `@error` эмитится наверх
9. При размонтировании — `<style>` и `<link>` теги чистятся

### Репозиторий, упомянутый модулем `custom_code`, раздаётся по HTTP

Шаг 4 — не внутренняя подробность: файлы репозитория читает **браузер
посетителя**, а не сервер. Это не то, что разработчик предполагает по умолчанию,
поэтому сказано прямо.

Кому и что отдаётся, решают четыре заслона в `backend/src/api/v2/modules/portal/`
(`code-guard.js` + обработчик в `router.js`), в этом порядке:

| заслон | код отказа | что решает |
|---|---|---|
| репозиторий назван конфигом | `REPO_NOT_IN_CONFIG` | слаг должен встретиться в `config.repo` хотя бы одного показываемого модуля `custom_code` |
| ref назван конфигом | `REF_NOT_ALLOWED` | читаются только ветки/теги из `config.ref` этих модулей (иначе — ветка репозитория по умолчанию); произвольный sha из истории не читается |
| роль видит модуль | `AUTH_REQUIRED` / `ROLE_REQUIRED` | доступ к исходнику повторяет видимость модуля на экране (`page.roles`, `mod.roles`, `mod.visible` — те же поля, что у `filterByRole`) |
| формат в белом списке | `FILE_TYPE_NOT_ALLOWED` | `BLOB_ALLOWED_EXTENSIONS`: исходники компонентов и статика. `.md`, `.csv`, `.sql`, `.xlsx`, файлы без расширения и `api/*.js` (серверные функции) наружу не идут |

**Решение считается на ОБРАЩЕНИЕ, а не на репозиторий целиком — иначе публичный
модуль открывает анониму файлы ролевого.** Так и было до 22.08.2026:
`collectRepoExposure` сводила `anonymous`, `roles` и `refs` по всему
репозиторию, и репозиторий, обслуживающий публичный модуль `main/widget.vue` и
ролевой (`roles: ['VIP']`, ветка `internal`), отдавал анониму
`internal/salary.vue` — ответ 200 с содержимым. Круг решающих модулей сужается
так:

- **по ветке — всегда**: браузер просит файлы по адресу `<repo>/blob/<ref>/`,
  то есть каждое обращение ветку называет (`codespaceBase` в
  `CustomCodeRuntime.vue`);
- **по имени файла — если оно названо входом хотя бы одного модуля этой ветки**;
  относительные импорты компонента (`./lib/utils.js`) конфигом не названы
  никогда, и решает им вся ветка: отказать значит погасить первый же
  многофайловый компонент.

Когда один файл назван и публичным модулем, и ролевым, он остаётся открыт
анониму: через публичный он на экране и так, а отказ погасил бы публичный
модуль. Строгость — в круге решающих модулей, не в пересечении их ролей.

**Удаление файла коммитом доступ НЕ закрывает, пока ref остался тем же:**
содержимое читается из истории, пока объект достижим. Чтобы убрать
засветившееся: `git filter-repo` по всей истории, затем в bare-репозитории на
сервере `git reflog expire --expire=now --all && git gc --prune=now`. Без
прунинга объект остаётся достижим по sha и после force-push.

Отсюда правило: **в репозитории портальных компонентов не держать ничего, кроме
кода компонентов.** Инструкции, выгрузки, дампы, токены — в другое место.

Что раздаётся прямо сейчас — считает `backend/scripts/codespace-blob-exposure.mjs`
(обходит все портальные конфиги, читает репозитории с диска той машины, где
запущен, и отличает «закрыто заслоном» от «файла нет»).

---

## Изоляция стилей

`<style>` блоки пользовательского компонента компилируются в браузере и добавляются
в `document.head`, то есть по устройству они глобальные. Чтобы правило вида
`.card { … }` внутри компонента не перекрашивало оболочку портала,
`CustomCodeRuntime.vue` рендерит обёртку `<div class="custom-code-root">`
(`display: contents`, в раскладке не участвует) и оборачивает каждый вставляемый
блок стилей в `@scope (.custom-code-root) { … }`.

### Какой лист заворачивается, а какой нет

Решение принимает `components/custom-code/styleIsolation.js` (чистые функции,
прогон — `node --test portal/components/custom-code/styleIsolation.test.mjs`).

| лист | обёртка | почему |
|---|---|---|
| `<style scoped>` | **нет** | уже ограничен атрибутом `data-v-…`, который компилятор дописывает к каждому селектору |
| `<style>` | да | по устройству глобален — ради него обёртка и заведена |
| лист с маркером `/* custom-code: global */` в начале | нет | автор объявил его глобальным намеренно |
| любой при `config.styleIsolation: 'off'` | нет | изоляция выключена на весь модуль |
| браузер без `@scope` | нет | Firefox ESR 140 и 115 |

**Про scoped — это противоположно первому впечатлению, и на этом мы обожглись.**
Обёртка не добавляет scoped-листу ничего, а телепортированные узлы отрезает:
атрибут `data-v` уезжает вместе с элементом в `<body>`, а корень области
видимости остаётся на месте. Замерено на проде 09.08.2026 (Firefox 152):
правило вида `.boxed[data-v-1a2b]` до узла в `<body>` **достаёт**, обёрнутое в
`@scope` правило — **нет**. До этой правки разметка внутри `Dialog` и `Popover`
на портале «Усадьбы» оставалась без собственных стилей.

### Когда нужен отказ от изоляции

За границу корня `@scope` не достаёт по устройству. Значит молча перестают
работать:

- `Teleport to="body"` и всё, что в нём;
- оверлеи PrimeVue — `.p-dialog`, `.p-popover`, `.p-menu`, `.p-datepicker`,
  `.p-select-overlay` (их PrimeVue тоже телепортирует);
- правила от классов на `<body>` (`body.dark-active .foo`);
- печатные стили, включая `@page`.

Два способа отказаться, по возрастанию охвата:

```css
/* один лист: маркер обязан стоять В НАЧАЛЕ, до первого правила */
/* custom-code: global */
.p-dialog { background: #1f2937; }
```

```jsonc
// весь модуль: в конфиге портала
{ "type": "custom_code", "config": { "styleIsolation": "off" } }
```

Маркер в середине листа изоляцию НЕ отменяет — иначе он повторил бы беду
прежней проверки на `@import`, которая срабатывала из любого места файла.

Остальное:

- **Наследование работает как раньше.** `@scope` ограничивает подбор селекторов, но не
  наследование: `--color-primary`, `--font-body`, `--border-radius` и прочие токены темы
  портала по-прежнему приходят от `.portal-layout`.
- **`@keyframes`, `@media`, `@supports` внутри `@scope` работают** — проверено в браузере.
- **`@import` и `@charset` выносятся отдельным тегом.** Внутри `@scope` они недопустимы,
  а допустимы только в начале таблицы. Прежде находка одного из них отменяла обёртку
  **всего листа** — то есть один `@import` молча снимал изоляцию целиком и работал
  необъявленной лазейкой. Теперь выносятся только они, остаток заворачивается.

Поддержка `@scope` определяется на лету (пробная таблица стилей, а не список версий).
Там, где её нет — Firefox ESR 140 и 115 — стили вставляются как раньше, без изоляции:
хуже, чем было, не становится, но и лучше тоже. Проверять именно пробным листом, а не
`CSS.supports('at-rule', '@scope')`: замерено 09.08.2026, что Firefox 152 отвечает на
этот вызов `false` при полностью работающем `@scope`.

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
| `Import not found: ./foo.vue` | Соседний файл не найден в том же codespace-репозитории на ветке `ref` | Закоммитьте файл; путь резолвится относительно текущего файла в репозитории |
| `Библиотека '@kit' не подключена: в конфиге модуля не задана версия` | В конфиге модуля нет поля `kit` | Добавьте `"kit": "<версия>"` (см. раздел «Библиотека `@kit`») |
| `Библиотека '@kit' версии X не найдена: 404` | Такая версия не выложена | Проверьте номер; выложенные версии перечислены в `portal-kit/README.md` |
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
