# Components

**Директория:** `components/`

## Основные компоненты

| Файл | Назначение |
|------|-----------|
| `PortalSkeleton.vue` | Skeleton-загрузчик для страниц портала |
| `ModuleRenderer.vue` | Рендерит модули страницы из конфига (`modules` массив). Диспетчер для `modules/` |
| `GalleryGrid.vue` | Сетка галереи с lazy loading изображений |
| `GuestCheckoutModal.vue` | Модал быстрого заказа без авторизации |
| `FluentEmoji.vue` | Fluent Emoji иконки |
| `ReportWidget.vue` | Встроенный виджет отчёта из backend |
| `ChatMessage.vue` | Пузырь сообщения AI-чата |
| `ChatWidget.vue` | Виджет AI-чата: кнопка-открывашка + панель с историей |

## modules/

Компоненты-модули, рендерятся через `ModuleRenderer.vue`. Каждый получает `module` prop с конфигом из `portal config`.

| Файл | Тип модуля | Назначение |
|------|-----------|-----------|
| `ModuleText.vue` | `text` | Текстовый блок / HTML-контент |
| `ModuleBanner.vue` | `banner` | Баннер с изображением и CTA |
| `ModuleFaq.vue` | `faq` | FAQ аккордеон |
| `ModuleMiniCatalog.vue` | `mini_catalog` | Мини-каталог: несколько товаров на странице |
| `ModuleReportWidget.vue` | `report_widget` | Виджет отчёта Integram |
| `ModuleCustomCode.vue` | `custom_code` | Кастомный код: Vue 3 SFC из codespace, компилируется в рантайме через vue3-sfc-loader. `<ClientOnly>` обёртка |

## custom-code/

| Файл | Назначение |
|------|-----------|
| `CustomCodeRuntime.vue` | Загрузка .vue из codespace API, компиляция через vue3-sfc-loader, рендер. Error boundary, style/CSS cleanup, `moduleCache` (Vue, vue-chartjs, chart.js), `compiledCache` (localStorage — кеш скомпилированного JS, ключ = хеш исходника) |
| `libraryRegistry.js` | Реестр опциональных CDN-библиотек. Большинство через esm.sh; Leaflet — исключение: CSS с unpkg.com, JS загружается через `<script>` тег (Firefox блокирует `dynamic import()` esm.sh для больших файлов). `LIBRARY_REGISTRY` для рантайма, `LIBRARY_CATALOG` для UI редактора |

## portal/

| Файл | Назначение |
|------|-----------|
| `ToolChips.vue` | Grouped tool chips with active/inactive toggle |
| `LiveDebatePanel.vue` | Real-time debate streaming panel with phase indicator |
| `AgentPicker.vue` | Agent selection modal with per-agent tool toggles |
| `DebateCard.vue` | Completed debate card: expert positions, cross-debate, synthesis |
| `TeamchatMessage.vue` | Chat message bubble with agent/human styling |

## panels/

| Файл | Назначение |
|------|-----------|
| `panels/PanelView.vue` | Панель управления рисками: список рисков с severity/stage, табы (Ситуация / Прогноз / Стратегия), интеграция с AI через `props.api.runAgent()`. Custom_code компонент (props: `api`, `bindings`, `db`) |

## Каталог — фильтрация по категориям

Страница каталога отображает category filter chips для фильтрации товаров. Категории загружаются через composable `useCategories` из `composables/usePortalData.js` (endpoint `GET /portal/api/catalog/categories`).

## layouts/portal.vue

Основной layout. Содержит:
- Хедер с навигацией (формируется из `config.pages`)
- `<slot />` для страниц
- `ChatWidget` (если включён в конфиге)
- Футер
