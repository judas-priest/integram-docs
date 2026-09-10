# Integram — зависимости

## Инфраструктура

| Сервис | Версия | Назначение | Заметки |
|--------|--------|------------|---------|
| **Node.js** | 24.x (минимум 18.x) | Runtime бэкенда | |
| **PostgreSQL** | 16.x | Основная БД, EAV-хранилище (`_v2_objects`, `"db"."db"`) | Заменила MariaDB |
| **Valkey** | 9.x | Job queue (BullMQ), кэш, pub/sub | Форк Redis, лицензия BSD, поддержка Linux Foundation |

### PostgreSQL extensions

| Расширение | Назначение |
|------------|------------|
| **pgvector** (`vector`) | Векторные эмбеддинги (1024d), HNSW-индексы для cosine search — граф, документы, память, KAG |
| **pg_trgm** | Триграммные индексы для LIKE/ILIKE по кириллице (поиск по EAV-значениям) |
| **pg_stat_statements** | Мониторинг производительности SQL-запросов |
| **pg_hint_plan** | Хинты для оптимизатора запросов |
| **TimescaleDB** | Гипертаблицы для временных рядов (`_v2_timeseries`, `_v2_portal_events`), автосжатие, `time_bucket`, `last()` — опциональный, fallback на обычные таблицы |

## Backend (Node.js / Express)

### Ядро

| Пакет | Назначение |
|-------|------------|
| `express` | HTTP-сервер, API роутинг |
| `pg` | Драйвер PostgreSQL |
| `pg-format` | Безопасное форматирование SQL-запросов |
| `bullmq` | Очередь задач (автоматизации, cron, delayed jobs) |
| `ws` | WebSocket — real-time обновления, совместное редактирование |
| `jsonwebtoken` | JWT-аутентификация |
| `@node-rs/bcrypt` | Хеширование паролей (нативная реализация, заменил `bcrypt`) |
| `cookie-parser` | Парсинг cookie |
| `basic-auth` | HTTP Basic Auth |

### AI / LLM

| Пакет | Назначение |
|-------|------------|
| `openai` | OpenAI-совместимый клиент (используется для KodaCode и других провайдеров) |
| `@anthropic-ai/sdk` | Anthropic Claude SDK |
| `@modelcontextprotocol/sdk` | MCP — протокол подключения внешних инструментов к агенту |

### Документы и файлы

| Пакет | Назначение |
|-------|------------|
| `quill-delta` | Операции над документами (CRDT-совместимый формат) |
| `mammoth` | Парсинг .docx |
| `pdf-parse` | Парсинг PDF |
| `xlsx` | Парсинг Excel/CSV |
| `carbone` | Генерация документов из шаблонов (PDF, DOCX) |
| `marked` | Markdown → HTML |
| `csv-parse` | Парсинг CSV-файлов |
| `fs-extra` | Расширенная работа с файловой системой |

### Безопасность и сеть

| Пакет | Назначение |
|-------|------------|
| `cors` | CORS-политики |
| `express-rate-limit` | Rate limiting API |
| `compression` | gzip-сжатие ответов |
| `nodemailer` | Отправка email |

### Утилиты

| Пакет | Назначение |
|-------|------------|
| `pino` / `pino-pretty` | Логирование (structured JSON) |
| `dotenv` | Переменные окружения |
| `zod` | Валидация схем данных |
| `axios` | HTTP-клиент (коннекторы, внешние API) |
| `multer` | Загрузка файлов |
| `js-yaml` | Парсинг/генерация YAML |
| `simple-git` | Работа с git-репозиториями |
| `isolated-vm` | V8-изоляция для пользовательских скриптов (script_button, run_script, workspace tools) |
| `p-limit` | Ограничение параллельных промисов |
| `@scalar/express-api-reference` | Swagger UI для OpenAPI-документации |

### Браузерная автоматизация

| Пакет | Назначение |
|-------|------------|
| `playwright` | Скрапинг, генерация скриншотов |
| `puppeteer` | Альтернативный браузерный движок |

## Frontend (Vue 3 / Vite)

### Ядро

| Пакет | Назначение |
|-------|------------|
| `vue` | UI-фреймворк |
| `vue-router` | Роутинг |
| `pinia` | State management |
| `vite` | Сборщик |
| `vite-ssg` | Static Site Generation поверх Vite |
| `tailwindcss` | Утилитарные CSS-стили |
| `tailwindcss-primeui` | Интеграция Tailwind с PrimeVue |
| `unplugin-vue-components` | Авто-импорт Vue-компонентов |
| `vite-plugin-pwa` | PWA (Service Worker, манифест) |
| `sass-embedded` | SCSS-препроцессор |

### UI-компоненты

| Пакет | Назначение |
|-------|------------|
| `primevue` | Библиотека UI-компонентов (таблицы, диалоги, формы) |
| `primeicons` | Иконки |
| `@primevue/themes` | Темы PrimeVue |
| `@primevue/auto-import-resolver` | Авто-импорт PrimeVue-компонентов |
| `chart.js` | Графики и диаграммы |
| `mermaid` | Диаграммы из текстового описания |
| `leaflet` | Карты |
| `gridstack` | Drag-and-drop сетка для дашбордов |
| `vue-draggable-plus` | Drag-and-drop для списков |

### Редактор документов

| Пакет | Назначение |
|-------|------------|
| `@tiptap/*` | Block-based редактор (расширение ProseMirror) |
| `quill` | Альтернативный редактор (legacy, для delta-совместимости) |
| `quill-cursors` | Курсоры совместного редактирования в Quill |
| `katex` | Рендеринг математических формул |
| `prismjs` | Подсветка синтаксиса кода |
| `dompurify` | Санитизация HTML |

### Визуализация графов

| Пакет | Назначение |
|-------|------------|
| `v-network-graph` | Визуализация графов (schema explorer, graph explorer) |
| `@dagrejs/dagre` | Автоматическая раскладка графов |

### Утилиты

| Пакет | Назначение |
|-------|------------|
| `date-fns` | Работа с датами (заменила `moment`) |
| `axios` | HTTP-клиент |
| `marked` | Markdown → HTML |
| `xlsx` | Экспорт/импорт Excel |
| `happy-dom` | DOM-эмулятор для тестов |

## Portal (Nuxt 3 / SSR)

Клиентский портал — отдельный SSR-фронтенд на Nuxt 3, запускается на порту 3000.

| Пакет | Назначение |
|-------|------------|
| `nuxt` | SSR-фреймворк на базе Vue 3 |
| `vue` | UI-фреймворк |
| `chart.js` | Графики и диаграммы |
| `vue-chartjs` | Vue-обёртка для Chart.js |
| `@nuxt/image` | Оптимизация изображений |
| `ipx` | Image processing (зависимость @nuxt/image) |
| `sharp` | Нативная обработка изображений |

## Тестирование

| Пакет | Сторона | Назначение |
|-------|---------|------------|
| `vitest` | Backend + Frontend | Unit/integration тесты |
| `@playwright/test` | Frontend | E2E тесты |
| `supertest` | Backend | HTTP-тесты API |
| `@vue/test-utils` | Frontend | Тесты Vue-компонентов |
| `jest` | Backend | Legacy-тесты (devDependency, не удалён) |

## Запуск (dev)

```bash
./dev-start.sh        # PostgreSQL + Valkey + Backend + Frontend + Portal
./dev-start.sh stop   # Остановить всё
```

Логи: `/tmp/nous-backend.log`, `/tmp/nous-frontend.log`, `/tmp/nous-portal.log`

