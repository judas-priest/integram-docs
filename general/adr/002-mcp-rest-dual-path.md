# ADR-002: MCP + REST — два входа для каждой фичи

**Статус:** accepted

## Контекст

Integram предоставляет AI-агента двумя способами: через REST API (in-app чат) и через MCP (Claude Code, внешние агенты). Оба должны выполнять одни и те же операции.

## Решение

Единая точка исполнения инструментов — `tool-executor.js`. Оба пути ведут в неё:

- **REST path**: `POST /api/v2/:db/ai/agent-chat` → `runner.js` → `tool-executor.js`
- **MCP path**: `mcp-server/index.js` → `POST /api/v2/:db/ai/tool` → `tool-executor.js`

MCP-сервер — тонкий прокси: он проксирует вызов инструмента на backend через HTTP, используя тот же `tool-executor.js`. Список инструментов динамически получается из `GET /api/v2/:db/ai/tools`.

Каталог инструментов — `TOOL_DEFS` в `agent/index.js` — единственный источник истины: описание, параметры, группа. При добавлении нового инструмента нужно:

1. Реализация в `tools/<group>.js`
2. Запись в `TOOL_DEFS`
3. `case` в `tool-executor.js`
4. Тир в `risk-tiers.js`

## Последствия

- Баг, исправленный только в REST-пути, остаётся в MCP — **фиксить оба пути**
- Тестировать нужно оба входа
- Новый инструмент = обязательно все 4 шага выше (checklist в `.claude/rules/ai-tools.md`)
- MCP HITL: `pending_confirmation` возвращается клиенту, который вызывает `confirm_action()`
