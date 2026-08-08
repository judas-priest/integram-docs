# Integram — Гайд для AI-агента / разработчика

Этот документ для Claude Code или другого AI-агента, который впервые работает с Integram.
Здесь — что это, как подключиться, что умеет, как работать.

---

## Что такое Integram

Единая среда для бизнеса: таблицы, документы, клиентский портал, AI-агенты, автоматизации, интеграции — всё в одной системе на общем слое данных.

**Ключевые сущности:**

| Сущность | Что это |
|---|---|
| **Workspace** | Изолированная база данных одного бизнеса/проекта. Всё живёт внутри workspace. |
| **Type (таблица)** | Структура данных — аналог таблицы в SQL. У каждого workspace свои типы. |
| **Requisite (колонка)** | Поле типа: текст, число, дата, bool, файл, ref (ссылка на другой тип), и т.д. |
| **Object (запись)** | Одна строка в таблице. Хранится в EAV-схеме (`_v2_objects` + `"db"."db"`). |
| **Portal** | Клиентский веб-портал. Строится из данных workspace. Отдельный Nuxt SSR (порт 3000). |

**EAV**: данные хранятся не в обычных таблицах, а в формате entity-attribute-value. Одна строка `_v2_objects` — одна запись. Значения полей — в таблице `"db"."db"`. Это важно, если работаешь напрямую с SQL.

---

## Два способа работы

### 1. MCP (рекомендуется для AI-агентов)

MCP-сервер превращает все 530+ инструментов Integram в MCP-инструменты. Работает через stdio.

**Подключение** — добавить в `claude_desktop_config.json` или аналог:

```json
{
  "mcpServers": {
    "integram": {
      "command": "node",
      "args": ["/path/to/integram/mcp-server/index.js"],
      "env": {
        "INTEGRAM_URL": "http://localhost:8081",
        "INTEGRAM_EMAIL": "user@example.com",
        "INTEGRAM_PASSWORD": "password",
        "INTEGRAM_WORKSPACE": "my-workspace"
      }
    }
  }
}
```

**Обязательный порядок работы через MCP:**

```
1. list_workspaces          — посмотреть доступные workspace
2. switch_workspace(slug)   — выбрать workspace (ОБЯЗАТЕЛЬНО перед любой работой)
3. search_tools("keyword")  — загрузить нужные инструменты (по умолчанию загружены только базовые)
4. <работать>
```

По умолчанию загружены только базовые CRUD-инструменты. Остальные группы активируются через `search_tools`:

| Запрос | Что активирует |
|---|---|
| `search_tools("schema")` | Создание/изменение таблиц и колонок, computed columns, AI-кнопки, валидация |
| `search_tools("reports")` | Отчёты с агрегацией |
| `search_tools("bulk")` | Массовые операции (bulk_create, bulk_update, bulk_delete) |
| `search_tools("documents")` | Документы, блоки, папки, версии |
| `search_tools("automations")` | Автоматизации и вебхуки |
| `search_tools("portal")` | Конфигурация и публикация клиентского портала |
| `search_tools("permissions")` | Роли и права доступа |
| `search_tools("graph")` | Граф связей между записями |
| `search_tools("workspace")` | Импорт/экспорт, бэкапы, файлы, дашборды |
| `search_tools("teamchat")` | Внутренний чат: комнаты, топики, сообщения |
| `search_tools("codespace")` | Git-репозитории: ветки, коммиты, PR |
| `search_tools("pm")` | Управление проектами: задачи, спринты, метрики |
| `search_tools("kag")` | Knowledge graph: поиск, импорт сущностей |
| `search_tools("memory")` | Долгосрочная память агента |

### 2. REST API

Все операции доступны через HTTP. Базовый URL: `http://localhost:8081/api/v2/`.

**Auth:**

```bash
# Получить токен
curl -X POST http://localhost:8081/api/v2/iam/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"password"}'
# → { "ok": true, "accessToken": "...", "refreshToken": "..." }

# Использовать в запросах
curl http://localhost:8081/api/v2/my-workspace/objects \
  -H "Authorization: Bearer <accessToken>"
```

**Формат ответа:**

```json
{ "ok": true, "data": { ... } }
{ "ok": false, "error": { "code": "NOT_FOUND", "message": "..." } }
```

**Все пути workspace — через `/:db/`:**

```
GET  /api/v2/:db/schema/types          — список таблиц
GET  /api/v2/:db/schema/types/:typeId  — структура таблицы (колонки)
GET  /api/v2/:db/objects               — записи (с фильтрацией, пагинацией)
POST /api/v2/:db/objects               — создать запись
PATCH /api/v2/:db/objects/:id          — обновить запись
DELETE /api/v2/:db/objects/:id         — удалить запись
```

Полный список: `GET /api/v2/openapi.json` или UI: `http://localhost:8081/api/v2/docs`

---

## Основные операции (MCP)

### Работа с данными

```
list_tables                     — все таблицы workspace
get_table_schema(typeId)        — колонки и их типы
list_objects(typeId, filters)   — записи таблицы
get_object(objId)               — одна запись
create_object(typeId, fields)   — создать запись
update_object(objId, fields)    — обновить
delete_object(objId)            — удалить (HITL!)
semantic_search(query)          — полнотекстовый + векторный поиск
```

**Значения полей:**
- Текст/число/дата/bool → передавать напрямую
- Ref (одиночный) → `ID` числом (из lookup-таблицы или связанной)
- Ref (мультиселект) → строка с именами через запятую: `"Тег 1, Тег 2"`

### Схема

```
search_tools("schema")  ← сначала это

create_table(name, icon)
add_column(typeId, alias, colTypeName)   — типы: text/number/date/bool/file/memo/ref/...
plan_schema({ tables: [...] })           — создать всю схему одним вызовом (рекомендуется)
```

`plan_schema` — самый мощный инструмент для создания схемы. Создаёт таблицы, колонки, связи и seed-данные за один вызов.

### Отчёты

```
search_tools("reports")  ← сначала это

create_report(name, parentTypeId)
add_report_column(reportId, columnAlias)
get_report(reportId)
```

### Портал

```
search_tools("portal")  ← сначала это

get_portal_config()                  — текущая конфигурация
set_portal_config(config)            — полная замена конфига (HITL!)
portal_publish(active: true/false)   — опубликовать/снять (HITL!)
get_portal_orders()
get_portal_catalog()
```

---

## AI-агент внутри Integram

Integram имеет собственного AI-агента (24 специализированных агента). К нему можно обращаться:

**Через чат** (SSE-стрим): `POST /api/v2/:db/ai/agent-chat`
**Через инструмент**: `POST /api/v2/:db/ai/tool` с `{ name, args }`

Агент понимает естественный язык, умеет создавать таблицы, заполнять данные, строить отчёты, запускать портал — всё через те же 530+ инструментов.

---

## HITL (Human-in-the-Loop)

Опасные операции (удаление, изменение схемы, публикация портала) требуют подтверждения.

**В MCP:** инструмент вернёт `"REQUIRES CONFIRMATION"` → спросить у пользователя → вызвать `confirm_action(approved=true/false)`.

**В REST:** `POST /:db/ai/mcp-resume` с `{ threadId, approved }`.

Без явного подтверждения пользователя не авто-подтверждать.

---

## Gotchas

- **EAV**: не ищи данные в обычных SQL-таблицах — всё в `_v2_objects` + `"db"."db"`. Используй инструменты, не сырой SQL.
- **Multiselect** = одна строка EAV, значения через запятую. Не отдельные строки на каждый тег.
- **Child tables** — записи дочерних таблиц создаются с `parentId`. В схеме связаны через `parentTypeId`.
- **`_value`** — виртуальная колонка с именем записи. Устанавливается через поле `name` при `create_object`. Не создавай колонку "Название" — она дублирует `_value`.
- **Config IDs в портале** — `typeId`/`reqId` никогда не хардкоди: всегда берутся из конфига портала.
- **Portal auth** — портальные пользователи (клиенты) ≠ workspace-пользователи. Разные JWT, разные auth-маршруты.

---

## Быстрый старт

Задача: создать CRM для магазина.

```
1. list_workspaces → switch_workspace("my-shop")
2. search_tools("schema")
3. plan_schema({
     tables: [
       { name: "Статусы", isLookup: true, seedRecords: ["Новый","В работе","Закрыт"] },
       { name: "Клиенты", columns: [
           { alias: "Телефон", type: "text" },
           { alias: "Email", type: "text" },
           { alias: "Статус", refTable: "Статусы" }
         ]
       },
       { name: "Заказы", columns: [
           { alias: "Клиент", refTable: "Клиенты" },
           { alias: "Сумма", type: "number" },
           { alias: "Дата", type: "datetime" },
           { alias: "Статус", refTable: "Статусы" }
         ]
       }
     ]
   })
4. create_object(typeId_Clients, { name: "Иван Иванов", "Телефон": "+79001234567" })
5. search_tools("reports") → create_report("Заказы по статусам", typeId_Orders)
```

---

## Ссылки

| Документ | Описание |
|---|---|
| [backend/docs/BACKEND.md](../backend/docs/BACKEND.md) | Полная документация backend: все модули, middleware, AI-агент |
| [docs/PORTAL.md](PORTAL.md) | Клиентский портал: конфигурация, модули, auth |
| [docs/deploy.md](deploy.md) | Развёртывание в production |
| `GET /api/v2/openapi.json` | Полная OpenAPI-спецификация (35 тегов, 200+ endpoints) |
| `GET /api/v2/docs` | Интерактивный UI (Scalar) |
