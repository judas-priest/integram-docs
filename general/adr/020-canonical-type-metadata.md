# ADR-020: Canonical Type Metadata — универсальный объектный слой

**Статус:** proposed
**Связано:** ADR-016 (объектный слой), ADR-007 (child tables), ADR-014 (inverted refs)

## Контекст

ADR-016 реализовал объектный слой как вертикальный пилот: тип `Product`, алиас
`ProductAlias`, identity keys `['gtin', 'article', 'sku']`. Код уже на 80%
параметризован (принимает `keyFields`, `canonicalTypeName`, `aliasTypeName`), но
конфигурация канонического типа **нигде не хранится** — каждый вызов должен знать,
какие ключи тождества, как нормализовать, где алиас-тип.

Проблемы текущей реализации:

1. **Коннектор не знает**, что целевой тип каноничен — `provenance` из пресета
   игнорируется, `source_pk` не сохраняется.
2. **Агент не знает**, какие типы канонические — не может автоматически применить
   upsert/resolve/movement без ручного конфига.
3. **normalizeKey** содержит `if (field === 'gtin')` — GTIN-специфичная логика
   зашита в код вместо конфига.
4. **Нет пути** от «развернул тип» до «коннектор автоматически пишет провенанс и
   дедуплицирует по ключам» — всё ручное.

## Решение

### 1. Canonical type metadata — JSONB на типе

Любой EAV-тип можно сделать каноническим, записав на нём метаданные:

```jsonc
{
  "canonical": true,
  "aliasTypeId": 42,
  "identityKeys": [
    { "field": "gtin", "normalize": "strip_leading_zeros", "unique": true },
    { "field": "article", "normalize": "lowercase" },
    { "field": "sku", "normalize": "lowercase" }
  ],
  "provenance": {
    // Колонки alias-типа, куда писать источник (уже есть по умолчанию)
    "roleField": "role",
    "sourceSystemField": "source_system",
    "sourcePkField": "source_pk"
  },
  "movementPatterns": [
    { "dir": "inbound", "re": "приход|поступлен|закуп|purchase|receipt" },
    { "dir": "outbound", "re": "продаж|отгруз|реализац|sale|shipment" }
  ]
}
```

### 2. Хранение

Satellite table `_v2_canonical_meta` (одна строка на тип):

```sql
CREATE TABLE IF NOT EXISTS _v2_canonical_meta (
  type_id    INTEGER PRIMARY KEY,   -- EAV type id
  alias_type_id INTEGER,            -- child alias type id
  config     JSONB NOT NULL DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

Почему отдельная таблица, а не JSONB-поле на EAV-строке типа:
- EAV `val` — текстовое поле, уже перегружено (имя типа + метаинфо иконок).
- Satellite таблицу можно JOIN-ить из коннектора одним запросом.
- Нет риска конфликта с другими EAV-операциями.
- `wsTable(db, '_v2_canonical_meta')` — стандартный паттерн проекта.

### 3. API (хелпер, не отдельный модуль)

Один файл `normalizer/canonical-meta.js`:

```js
/** Читает canonical-конфиг типа. Null = тип не канонический. */
export async function getCanonicalMeta(pool, db, typeId)

/** Записывает/обновляет canonical-конфиг (idempotent UPSERT). */
export async function setCanonicalMeta(pool, db, typeId, aliasTypeId, config)

/** Все канонические типы воркспейса (для агента). */
export async function listCanonicalTypes(pool, db)

/** Удаляет canonical-мету (тип перестаёт быть каноническим). */
export async function removeCanonicalMeta(pool, db, typeId)
```

### 4. Точки интеграции

#### 4.1. ensureCanonicalTypes (setup)

После создания типов — автоматически `setCanonicalMeta`:

```js
// object-layer-setup.js — в конце ensureCanonicalTypes():
await setCanonicalMeta(pool, db, productTypeId, aliasTypeId, {
  canonical: true,
  identityKeys: canonicalColumns
    .filter(c => c.unique || DEFAULT_KEY_FIELDS.includes(c.alias))
    .map(c => ({ field: c.alias, normalize: c.normalize || 'lowercase' })),
  movementPatterns: config.movementPatterns || null,
});
```

#### 4.2. connectors/service.js (import)

При import в тип, у которого есть canonical meta:

```
executeConnector():
  1. targetTypeId задан
  2. meta = await getCanonicalMeta(pool, db, targetTypeId)
  3. if (meta):
     a. При CREATE/UPDATE — записать source_system, source_pk из конфига
        коннектора (config.provenance || preset.provenance)
     b. Если identity keys есть — вызвать findCanonicalByKey() с meta.identityKeys
        вместо стандартного idField-dedup
     c. При merge — создать/обновить alias child-запись
```

Это **опциональный** путь: если у типа нет canonical meta, коннектор работает
как раньше. Каноничность — opt-in.

#### 4.3. object-layer-upsert.js (identity)

Читает ключи из meta вместо `DEFAULT_KEY_FIELDS`:

```js
export async function upsertCanonicalBatch(pool, db, records, opts) {
  // opts.meta || await getCanonicalMeta(pool, db, opts.productTypeId)
  const meta = opts.meta || await getCanonicalMeta(...);
  const keyFields = (meta?.config?.identityKeys || []).map(k => k.field);
  // ... остальная логика без изменений
}
```

`normalizeKey` читает правило из `identityKeys[i].normalize`:

```js
const NORMALIZERS = {
  strip_leading_zeros: v => v.replace(/^0+/, '') || v,
  lowercase: v => v.toLowerCase(),
  trim: v => v.trim(),
  uppercase: v => v.toUpperCase(),
  noop: v => v,
};

function normalizeKey(field, value, identityKeys) {
  if (!value) return null;
  const v = String(value).trim();
  if (!v) return null;
  const rule = identityKeys?.find(k => k.field === field);
  const fn = NORMALIZERS[rule?.normalize || 'lowercase'];
  return fn(v);
}
```

#### 4.4. AI agent — TOOL_DEFS

Новый инструмент `list_canonical_types` (read-only, risk=low):

```js
{
  name: 'list_canonical_types',
  description: 'Список канонических типов воркспейса с identity keys и alias-типами.',
  parameters: {},
  returns: [{ typeId, typeName, aliasTypeId, aliasTypeName, identityKeys }]
}
```

Существующие O-A1/O-A2/O-A3 инструменты (resolve_aliases, resolve_identity,
get_canonical_movement) регистрируются в TOOL_DEFS и читают мету автоматически —
не требуют ручного `productTypeId`.

#### 4.5. connector provenance

Коннектор записывает source_system и source_pk при каждом import:

```
Connector config (или preset):
  provenance: { source_system: '1c-buh', role_hint: 'bookkeeping' }

При import каждой записи:
  1. source_system = config.provenance.source_system
  2. source_pk = idValue (внешний PK записи)
  3. role_hint = config.provenance.role_hint

Эти данные передаются в resolveAliasesTool / upsertCanonicalBatch.
```

## Что НЕ меняется

- EAV-модель объектов — без изменений.
- Формат child-таблицы алиасов (role, source_system, source_pk) — без изменений.
- resolveAliasesTool, resolveIdentityTool, getCanonicalMovementTool — код тот же,
  только читают meta вместо дефолтов.
- Существующие коннекторы без canonical-типа — работают как раньше.

## Объем изменений

| Файл | Изменение | Строк |
|------|-----------|-------|
| `normalizer/canonical-meta.js` | Новый: 4 функции CRUD + ensure table | ~60 |
| `normalizer/object-layer-setup.js` | +setCanonicalMeta после создания типов | ~5 |
| `normalizer/object-layer-upsert.js` | Читать keys из meta; NORMALIZERS; rename -> upsertCanonicalBatch | ~20 |
| `normalizer/object-layer.js` | Читать meta вместо fallback 'product' | ~15 |
| `connectors/service.js` | При import: getCanonicalMeta -> provenance + identity keys | ~25 |
| `ai/agent/index.js` | +4 TOOL_DEFS (list_canonical_types + 3 existing tools) | ~40 |
| `ai/agent/tool-executor.js` | +case для 4 новых tools | ~20 |
| Тесты | canonical-meta CRUD, upsert с meta, connector provenance | ~80 |
| **Итого** | | **~265** |

## Миграция

Для существующих воркспейсов с Product/ProductAlias (пилот ADR-016):

```js
// Одноразовый скрипт или в ensureCanonicalTypes при первом запуске:
// Если тип Product есть, но meta нет — создать meta из дефолтов.
```

Обратная совместимость: `upsertProductBatch` -> deprecated alias на `upsertCanonicalBatch`.
`findProductByKey` -> deprecated alias на `findCanonicalByKey` (уже есть).

## Примеры каноничных типов

### Product (пилот, ADR-016)
```jsonc
{
  "identityKeys": [
    { "field": "gtin", "normalize": "strip_leading_zeros", "unique": true },
    { "field": "article", "normalize": "lowercase" },
    { "field": "sku", "normalize": "lowercase" }
  ],
  "movementPatterns": [
    { "dir": "inbound", "re": "приход|поступлен|закуп" },
    { "dir": "outbound", "re": "продаж|отгруз|реализац" }
  ]
}
```

### Contractor (следующий вертикал)
```jsonc
{
  "identityKeys": [
    { "field": "inn", "normalize": "strip_leading_zeros", "unique": true },
    { "field": "ogrn", "normalize": "noop", "unique": true },
    { "field": "name", "normalize": "lowercase" }
  ],
  "movementPatterns": [
    { "dir": "inbound", "re": "поставк|закуп|оплата входящ" },
    { "dir": "outbound", "re": "продаж|реализац|оплата исходящ" }
  ]
}
```

### Employee (кросс-система)
```jsonc
{
  "identityKeys": [
    { "field": "snils", "normalize": "strip_non_digits", "unique": true },
    { "field": "inn", "normalize": "strip_leading_zeros" },
    { "field": "tab_number", "normalize": "lowercase" }
  ]
}
```

## Последствия

**Плюсы:**
- Любой тип становится каноническим одной записью meta — без нового кода.
- Коннектор автоматически пишет провенанс и дедуплицирует по identity keys.
- Агент видит все канонические типы через `list_canonical_types`.
- normalizeKey конфигурируемый — нет `if (field === 'gtin')` в коде.
- Обратная совместимость: пилот Product продолжает работать.

**Минусы:**
- Одна новая satellite table на воркспейс.
- Коннектор делает один лишний SELECT при import (getCanonicalMeta) — кешируется
  на время batch.

**Риски:**
- Identity keys должны быть существующими колонками типа — валидировать при
  setCanonicalMeta.
- movementPatterns regex компилируется при каждом вызове — кешировать при чтении meta.
