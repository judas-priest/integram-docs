# Architecture Decision Records

Короткие документы "почему мы так сделали". Читай перед тем, как менять архитектуру.

## Каталогов ADR два, нумерация независимая

| каталог | что там | канонический |
|---|---|---|
| `docs/adr/` | общеплатформенные решения | **да**, на него указывает `CLAUDE.md` |
| `backend/docs/adr/` | решения внутри бэкенда | нет |

Отсюда следствие для читателя и для автора ссылки: **запись вида «ADR-0NN» без
указания каталога неоднозначна.** Конкретно:

- **010 занят в обоих каталогах под разные темы:**
  `docs/adr/010-ws-subscribe-authorization.md` (авторизация подписки WS) и
  `backend/docs/adr/010-workspace-tools.md` (инструменты уровня воркспейса).
- **009 занят только в бэкендовом каталоге** (`009-a2a-interactive-flows.md`);
  в этом каталоге номера 009 нет.
- **015 не занят нигде — и на него ссылаются.** `docs/adr/016-object-layer.md`
  (строки 69 и 140) и `docs/adr/018-client-source-resolution.md` (строка 86)
  ссылаются на «ADR-015 (ревью/акты), модель ревью-ворот». Документ никогда не был
  написан; ссылка висячая. Занимать номер 015 чем-то другим нельзя: тогда две
  живые ссылки станут указывать на чужой документ, а это хуже висячей ссылки,
  потому что разрешается. Разбор — в [ADR-026](026-presence-and-last-seen.md),
  раздел «Почему номер 026, а не 015».

## Индекс

| # | Решение | Статус |
|---|---------|--------|
| [001](001-eav-schema.md) | EAV-схема как основа хранилища | accepted |
| [002](002-mcp-rest-dual-path.md) | MCP + REST — два входа для каждой фичи | accepted |
| [003](003-multiselect-storage.md) | Хранение multiselect в EAV | accepted |
| [004](004-router-service-separation.md) | router.js — HTTP only, service.js — бизнес-логика | accepted |
| [005](005-event-bus.md) | Event bus для межмодульного взаимодействия | accepted |
| [006](006-portal-auth-separate.md) | Portal JWT отдельно от workspace JWT | accepted |
| [007](007-child-tables-vs-refs.md) | Child tables (up=parentId) vs ref-колонки | accepted |
| [008](008-multi-agent-orchestration.md) | Паттерн мультиагентной оркестрации | accepted |
| — | *009 — нет; номер занят в `backend/docs/adr/`* | |
| [010](010-ws-subscribe-authorization.md) | WebSocket subscribe-level authorization | accepted |
| [011](011-calls-pure-p2p.md) | Calls — чистый P2P mesh без SFU/медиа-сервера | accepted |
| [012](012-hitl-agent-suggestion-from-behavior.md) | HITL-предложение агентов из поведенческих паттернов | accepted |
| [013](013-module-owns-ddl.md) | Модуль владеет своим DDL — bootstrap не дублирует | accepted |
| [014](014-eav-ref-inverted-techdebt.md) | EAV ref storage — inverted pattern techdebt | accepted |
| — | *015 — не занят, но на него ссылаются ADR-016 и ADR-018 (см. выше)* | |
| [016](016-object-layer.md) | Объектный слой — канонические объекты поверх разнородных источников | proposed |
| [017](017-agent-runtime.md) | Рантайм агента — заземлённое действие по правилам с правом отказа | proposed |
| [018](018-client-source-resolution.md) | Резолюция клиентских данных — golden record + governed writeback | proposed |
| [019](019-executable-data-specs.md) | Декларативный слой исполняемых спеков данных | proposed |
| [020](020-canonical-type-metadata.md) | Canonical Type Metadata — универсальный объектный слой | proposed |
| [021](021-remote-workspace-db.md) | Remote workspace databases — данные воркспейса на отдельном PG-сервере | accepted |
| [022](022-sandbox-profiles.md) | Профили песочницы — одно объявление способностей, пределов и таймаутов | accepted |
| [023](023-automation-server-function-action.md) | Вызов серверной функции codespace как действие автоматизации | accepted |
| [024](024-workspace-defined-agents.md) | Агент, заданный данными воркспейса | accepted |
| [025](025-agent-knowledge-scope.md) | Область знаний — поле агента | accepted |
| [026](026-presence-and-last-seen.md) | Присутствие и «последний визит» — две разные вещи, два хранилища | accepted |

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

Перед тем как взять номер: посмотреть **оба** каталога и таблицу выше. Номер,
свободный в `docs/adr/`, может быть занят в `backend/docs/adr/`, а свободный в
обоих — уже упомянут ссылкой из существующего документа.
