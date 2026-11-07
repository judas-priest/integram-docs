# ADR-003: Хранение multiselect в EAV

**Статус:** accepted

## Контекст

Колонка с мультивыбором может хранить несколько значений. В EAV есть два технических способа: несколько строк в таблице или одна строка с разделителем. Выбор влияет на все операции чтения, записи и фильтрации.

## Решение

Поведение зависит от типа колонки. Признак мультивыбора — модификатор `:MULTI:` в `val` определения колонки (`modifiers.js`).

**Lookup multiselect** (`:MULTI:` + base type, т.е. ссылка на колонку-справочник типа SHORT):

- Хранится в **одной строке EAV**: `t=colDefId, val="18,21"` (comma-separated IDs)
- При записи: `String(value)` передаётся as-is, `formatVal` не применяется (иначе "18,21" → "18")
- При чтении: строка `"18,21"`, фронтенд делает `split(',')`
- MCP: принимает `"urgent, important"` (текстовые имена через запятую), backend резолвит в ID

**Ref multiselect** (`:MULTI:` + user-defined type, ссылка на другую EAV-таблицу):

- Хранится в **нескольких строках EAV**: каждая `t=refObjId, val=String(colDefId), ord=i`
- При записи массива: DELETE старых + INSERT новых
- При чтении: строки агрегируются в массив по `val=String(colDefId)`
- Для каждой ref создаётся/удаляется ребро в graph (`object.edge.created/deleted`)

**Single-ref** (без `:MULTI:`):

- Одна строка: `t=refObjId, val=String(colDefId)`

## Последствия

- Запрещено хранить multiselect как несколько строк с одинаковым `t=colDefId` — это сломает чтение lookup multiselect
- При получении массива для single-ref колонки backend берёт только последний элемент и логирует warning
- Фронтенд всегда split/join на `,` для lookup multiselect, никогда не передаёт массив напрямую
- Фильтрация lookup multiselect через LIKE/contains, не через JOIN
