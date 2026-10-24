# ADR-040: Typed tools внешних агентов — нативный каталог, MCP-мост отложен

**Статус:** accepted

## Контекст

Внешние агенты реестра `_v2_agent_registry` вызываются только через
`delegate_to_agent(slug, task)` — текстовая задача → текстовый результат. Колонка
`tools` у external-агентов мертва. У internal-агентов та же колонка хранит перечень
ИМЁН существующих инструментов (механизм ADR-024-workspace-defined-agents.md) —
у колонки два значения, валидатор обязан это различать.

## Решение

1. `type='external'` держит в `tools` массив определений
   `{name, description, parameters: JSON Schema}`; `type='internal'` — по-прежнему
   имена. Форму проверяет `normalizeTypedTools()` в service, допуск на
   HTTP-границе — Zod.
2. Discovery: `GET /:db/ai/tools` — четвёртый источник `listTypedTools()`, имена
   `ag_<slug>__<tool>`, разбор по ПОСЛЕДНЕМУ `__` (слаг может содержать `_`,
   в имени инструмента `__` запрещено). MCP получает их автоматически
   (`fetchTools` → тот же маршрут, EN-описание падает на описание тула).
3. Вызов: `POST /:db/ai/tool` и путь агента → ветка диспетчера по префиксу →
   `invoke(slug, {tool, args})` → POST callback телом
   `{taskId, tool, args, context, workspace}`; `task` и `tool` взаимоисключающи;
   только `protocol='integram'` (a2a-тело задачи не знает).
4. Тир TIER_HIGH (семантика «irreversible or external»), роль 'admin'
   (паритет `delegate_to_agent`). Подтверждения HITL НЕТ: `HITL_TOOL_SET`
   статический, pending-запись пишет туловый кейс, которого здесь нет;
   прецедент — `ncl_*` (middleware.js:73-77). Гейт automation-режима в executeTool читает
   `TOOL_TIERS` напрямую — typed tools им не блокируются; уровень доверия тот
   же, что у `delegate_to_agent` (MEDIUM, в автоматизации разрешён).
5. Валидация при регистрации: имя `[a-z0-9_]{1,64}` без `__`, description
   1-2000, `parameters.type === 'object'`, ≤ 50 на агента, дубликаты запрещены.

## Цена (названа вслух)

- Чужие JSON Schema попадают в общий каталог модели. Схемы не исполняются
  платформой — довод до исполнения идёт через `validateOutboundUrl` + `safeFetch`
  на каждом `invoke` (перевалидация URL живая).
- Язык описаний не контролируем — агент присылает description как есть.
- TIER_HIGH без HITL-подтверждения (см. п. 4) — не ошибка классификатора, а
  зафиксированный уровень гарантий.
- Спецправило префикса в `toolRiskTier`/`minRoleForTool` — вторая точка знания
  о внешних инструментах после `listTypedTools`.

## Почему не MCP-мост

У backend нет MCP-клиента (`mcp-server/` — исходящий прокси в обратную сторону:
клиенты → наш REST). Полный MCP-сервер на стороне каждого агента несоразмерен
для триггера «2+ агента»; метрики (`_v2_agent_tasks`), HMAC и health пришлось бы
строить заново поверх MCP-транспорта. Дверь к B: колонка `protocol` — добавляется
значение `'mcp'` и отдельный ADR на транспорт; вариант A ничего не блокирует.

## Ссылки

ADR-024-workspace-defined-agents.md (родственный механизм для internal),
002-mcp-rest-dual-path.md (оба входа), docs/tech-debt.md (TD-015).
