# Architecture Decision Records

Короткие документы "почему мы так сделали". Читай перед тем, как менять архитектуру.

## Индекс

| # | Решение |
|---|---------|
| [001](001-eav-schema.md) | EAV-схема как основа хранилища |
| [002](002-mcp-rest-dual-path.md) | MCP + REST — два входа для каждой фичи |
| [003](003-multiselect-storage.md) | Хранение multiselect в EAV |
| [004](004-router-service-separation.md) | router.js — HTTP only, service.js — бизнес-логика |
| [005](005-event-bus.md) | Event bus для межмодульного взаимодействия |
| [006](006-portal-auth-separate.md) | Portal JWT отдельно от workspace JWT |
| [007](007-child-tables-vs-refs.md) | Child tables (up=parentId) vs ref-колонки |
| [008](008-multi-agent-orchestration.md) | Паттерн мультиагентной оркестрации |
| [010](010-ws-subscribe-authorization.md) | WebSocket subscribe-level authorization |
| [011](011-calls-pure-p2p.md) | Calls — чистый P2P mesh без SFU/медиа-сервера |
| [012](012-hitl-agent-suggestion-from-behavior.md) | HITL-предложение агентов из поведенческих паттернов |
| [013](013-module-owns-ddl.md) | Модуль владеет своим DDL — bootstrap не дублирует |
| [021](021-remote-workspace-db.md) | Remote workspace databases — данные воркспейса на отдельном PG-сервере |
| [022](022-sandbox-profiles.md) | Профили песочницы — одно объявление способностей, пределов и таймаутов |
| [023](023-automation-server-function-action.md) | Вызов серверной функции codespace как действие автоматизации |

## Шаблон для нового ADR

```markdown
# ADR-NNN: Название

**Статус:** accepted | deprecated | superseded by ADR-NNN

## Контекст
Почему нужно было принять решение.

## Решение
Что именно решили.

## Последствия
- Что нельзя делать иначе и почему
- Что упрощается
- Что усложняется
```
