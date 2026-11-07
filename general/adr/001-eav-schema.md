# ADR-001: EAV-схема как основа хранилища

**Статус:** accepted

## Контекст

Integram — no-code платформа, где пользователи сами создают таблицы и колонки. Количество и типы колонок заранее неизвестны, меняются без деплоя. Классическая реляционная схема потребовала бы `ALTER TABLE` при каждом изменении схемы пользователя, что недопустимо в multi-tenant среде.

## Решение

Используется Entity-Attribute-Value (EAV) схема. Каждый workspace — отдельная PostgreSQL-схема. Внутри неё **одна** основная EAV-таблица `"db"."db"` с колонками:

```sql
id  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
up  BIGINT NOT NULL DEFAULT 0,   -- parentId или 0 для корневых
ord INT    NOT NULL DEFAULT 1,
t   BIGINT NOT NULL DEFAULT 0,   -- typeId или refObjId (для ref-колонок)
val TEXT                          -- хранимое значение
```

В этой **одной** таблице хранится всё:
- **Объекты** (строки): `up=parentId, t=typeId, val=displayName(_value)`
- **Определения типов** (таблиц): `up=0, t=базовый_тип_или_parentTypeId, val=typeName`
- **Определения колонок**: `up=typeId, t=baseType, val=модификаторы(:ALIAS=...:!NULL::MULTI:)+baseTypeName`
- **Значения базовых реквизитов**: `up=objectId, t=reqTypeId, val=хранимое_значение`
- **Значения ref-реквизитов**: `up=objectId, t=refObjId, val=String(colDefId)`

Отдельных таблиц для реквизитов нет — всё в `"db"."db"`.

Удалённые объекты перемещаются в satellite-таблицу `_v2_trash` (отдельная структура, не флаг в EAV).

Доступ к таблице — только через `eavTable(db)` из `shared/sql-guards.js`, который возвращает `"db"."db"`. Никогда хардкодом.

## Последствия

- Схема пользователя меняется без SQL-миграций: добавить колонку = добавить запись в EAV
- Multi-tenancy через изоляцию PostgreSQL-схем, а не строки-фильтры
- Запросы сложнее реляционных: разграничение объектов и реквизитов — по `up` и `t`, не по таблице
- Все запросы параметризованы (`?`), конкатенация SQL строк запрещена
- `SELECT id, val FROM ${eavTable(db)} WHERE id = ?` — читает _value объекта
- `SELECT up, t, val FROM ${eavTable(db)} WHERE up IN (?)` — читает реквизиты
- Computed columns (LOOKUP, ROLLUP, FORMULA) вычисляются при чтении, не хранятся
