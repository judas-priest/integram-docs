# Custom Code — примеры компонентов

Готовые примеры кастомных Vue 3 SFC компонентов для портала. Каждый можно скопировать в codespace-репозиторий и подключить через визуальный редактор.

Основная документация: [custom-code.md](custom-code.md)

---

## 1. Калькулятор стоимости услуг

Форма с чекбоксами услуг, автоматическим расчётом итоговой суммы и отправкой заявки в таблицу воркспейса.

### Требования

**Привязки (bindings):**

| Ключ | Формат | Описание |
|------|--------|----------|
| `typeId` | `table:ID` | Таблица услуг (с колонкой `price`) |
| `orderTypeId` | `table:ID` | Таблица заказов (куда создаётся запись) |

**Библиотеки:** не требуются (только встроенные Vue 3)

### Код

```vue
<template>
  <div class="calculator">
    <h2>Калькулятор стоимости</h2>
    <div v-for="s in services" :key="s.id" class="service-row">
      <input type="checkbox" :value="s.id" v-model="selected" />
      <span>{{ s._value }}</span>
      <span class="price">{{ s.price }} ₽</span>
    </div>
    <hr />
    <div class="total">Итого: <b>{{ total }} ₽</b></div>
    <button class="btn btn-primary" @click="submit" :disabled="submitting">
      {{ submitting ? 'Отправка...' : 'Оформить заявку' }}
    </button>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

const props = defineProps(['bindings', 'api', 'db'])
const selected = ref([])
const submitting = ref(false)

// Реактивная коллекция — кешируется, дедуплицируется между модулями
const servicesData = props.api.useCollection(props.api.bindings?.typeId?.id)
const services = computed(() => servicesData.value?.items || [])

const total = computed(() =>
  services.value
    .filter(s => selected.value.includes(s.id))
    .reduce((sum, s) => sum + Number(s.price || 0), 0)
)

async function submit() {
  submitting.value = true
  try {
    await props.api.createRecord(props.api.bindings?.orderTypeId?.id, {
      name: `Заказ ${new Date().toLocaleDateString()}`,
      fields: { 'Сумма': total.value, 'Услуги': selected.value.join(',') }
    })
    props.api.toast.success('Заявка отправлена!')
    selected.value = []
  } catch (e) {
    props.api.toast.error(e.message)
  } finally {
    submitting.value = false
  }
}
</script>

<style scoped>
.calculator { padding: 2rem; }
.service-row { display: flex; align-items: center; gap: 1rem; padding: 0.5rem 0; }
.price { margin-left: auto; font-weight: 600; }
.total { font-size: 1.5rem; margin: 1rem 0; }
.btn { padding: 0.75rem 1.5rem; border: none; border-radius: var(--border-radius); cursor: pointer; }
.btn-primary { background: var(--color-primary); color: #fff; }
.btn:disabled { opacity: 0.6; cursor: not-allowed; }
</style>
```

### Как это работает

1. `api.useCollection(typeId)` загружает список услуг из EAV-таблицы. Результат кешируется в `PortalDataLayer` -- повторные вызовы с тем же `typeId` не создают новых запросов.
2. Пользователь отмечает нужные услуги чекбоксами. Computed-свойство `total` автоматически пересчитывает сумму.
3. При нажатии кнопки `api.createRecord` отправляет POST-запрос на `/portal/api/objects`, создавая новую EAV-запись в таблице заказов.
4. Toast-уведомление показывает результат через `api.toast.success/error`.

### Пример конфига модуля

```json
{
  "type": "custom_code",
  "config": {
    "repo": "portal-components",
    "file": "calculator.vue",
    "ref": "main",
    "libraries": [],
    "bindings": {
      "typeId": "table:42",
      "orderTypeId": "table:55"
    }
  }
}
```

---

## 2. Дашборд с графиками

KPI-карточки с ключевыми метриками и столбчатая диаграмма из данных отчёта.

### Требования

**Привязки (bindings):**

| Ключ | Формат | Описание |
|------|--------|----------|
| `reportId` | `report:ID` | ID отчёта с данными для KPI и графика |

**Библиотеки:** не требуются (vue-chartjs и chart.js встроены)

### Код

```vue
<template>
  <div class="dashboard">
    <h2 :style="{ fontFamily: 'var(--font-heading)' }">Аналитика</h2>
    <div class="grid">
      <div v-for="kpi in kpis" :key="kpi.label" class="card kpi-card">
        <div class="kpi-value">{{ kpi.value }}</div>
        <div class="kpi-label">{{ kpi.label }}</div>
      </div>
    </div>
    <div class="card" v-if="chartData">
      <div class="card-body">
        <Bar :data="chartData" :options="chartOptions" />
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'
import { Bar } from 'vue-chartjs'
import { Chart, BarElement, CategoryScale, LinearScale, Tooltip } from 'chart.js'

Chart.register(BarElement, CategoryScale, LinearScale, Tooltip)

const props = defineProps(['bindings', 'api', 'db'])
const chartOptions = { responsive: true }

// Реактивный отчёт — кешируется, дедуплицируется
const reportData = props.api.useReport(props.api.bindings?.reportId?.id)

const kpis = computed(() => {
  const { columns, rows } = reportData.value || { columns: [], rows: [] }
  return rows.slice(0, 4).map(r => ({
    label: r[columns[0]?.alias],
    value: r[columns[1]?.alias]
  }))
})

const chartData = computed(() => {
  const { columns, rows } = reportData.value || { columns: [], rows: [] }
  if (!columns.length) return null
  return {
    labels: rows.map(r => r[columns[0].alias]),
    datasets: [{
      label: columns[1].alias,
      data: rows.map(r => Number(r[columns[1].alias])),
      backgroundColor: getComputedStyle(document.documentElement)
        .getPropertyValue('--color-primary').trim()
    }]
  }
})
</script>

<style scoped>
.dashboard { padding: 2rem; }
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1rem;
  margin-bottom: 2rem;
}
.kpi-card { padding: 1.5rem; text-align: center; }
.kpi-value { font-size: 2rem; font-weight: 700; color: var(--color-primary); }
.kpi-label { color: var(--color-text-secondary); margin-top: 0.25rem; }
.card {
  background: var(--color-surface);
  border: var(--card-border);
  border-radius: var(--border-radius);
  box-shadow: var(--card-shadow);
}
.card-body { padding: 1.5rem; }
</style>
```

### Как это работает

1. `api.useReport(reportId)` загружает данные отчёта. Отчёт возвращает `{ columns, rows }` -- массив колонок с алиасами и массив строк.
2. Первые 4 строки отчёта отображаются как KPI-карточки. Первая колонка -- название метрики, вторая -- значение.
3. Все строки отображаются на столбчатой диаграмме (Bar) через vue-chartjs. Цвет столбцов берётся из CSS-переменной `--color-primary` портала.
4. Стили карточек используют CSS-переменные портала (`--color-surface`, `--card-border`, `--card-shadow`), чтобы визуально соответствовать выбранному пресету.

### Пример конфига модуля

```json
{
  "type": "custom_code",
  "config": {
    "repo": "portal-components",
    "file": "dashboard.vue",
    "ref": "main",
    "libraries": [],
    "bindings": {
      "reportId": "report:12"
    }
  }
}
```

---

## 3. Конфигуратор товара

Интерактивный конфигуратор с выбором опций (цвет, размер и т.д.) и динамическим пересчётом цены на основе модификаторов из отчёта.

### Требования

**Привязки (bindings):**

| Ключ | Формат | Описание |
|------|--------|----------|
| `productSlug` | `raw` | Slug товара из каталога |
| `optionsReportId` | `report:ID` | Отчёт с опциями (колонки: `name`, `values` через запятую) |
| `modifiersReportId` | `report:ID` | Отчёт с ценовыми модификаторами (колонки: `option`, `value`, `extra`) |

**Библиотеки:** не требуются

### Код

```vue
<template>
  <div class="configurator" v-if="product">
    <div class="preview">
      <img :src="currentPhoto" :alt="product._value" />
    </div>
    <div class="options">
      <h2>{{ product._value }}</h2>
      <div v-for="opt in options" :key="opt.name" class="option-group">
        <label>{{ opt.name }}</label>
        <div class="option-buttons">
          <button
            v-for="val in opt.values" :key="val"
            :class="['option-btn', { active: choices[opt.name] === val }]"
            @click="choose(opt.name, val)"
          >{{ val }}</button>
        </div>
      </div>
      <hr />
      <div class="price">{{ calculatedPrice }} ₽</div>
      <button class="btn btn-primary" @click="addToCart">В корзину</button>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, computed } from 'vue'

const props = defineProps(['bindings', 'api', 'db'])
const choices = reactive({})

// Реактивные данные — кешируются, дедуплицируются
const productData = props.api.useRecord(props.api.bindings?.productSlug?.raw)
const product = computed(() => productData.value)
const currentPhoto = computed(() => product.value?.fields?.photo || '')

// Опции и модификаторы из отчётов (если привязаны)
const optionsReport = props.api.bindings?.optionsReportId?.id
  ? props.api.useReport(props.api.bindings?.optionsReportId?.id)
  : ref(null)
const modifiersReport = props.api.bindings?.modifiersReportId?.id
  ? props.api.useReport(props.api.bindings?.modifiersReportId?.id)
  : ref(null)

const options = computed(() => {
  const rows = optionsReport.value?.rows || []
  return rows.map(r => ({
    name: r.name,
    values: r.values?.split(',').map(v => v.trim()) || []
  }))
})

const priceModifiers = computed(() => modifiersReport.value?.rows || [])

function choose(optName, val) {
  choices[optName] = val
}

// Цена вычисляется реактивно — data-driven через модификаторы из отчёта
const calculatedPrice = computed(() => {
  const base = Number(product.value?.fields?.price || 0)
  const extra = priceModifiers.value
    .filter(m => choices[m.option] === m.value)
    .reduce((sum, m) => sum + Number(m.extra || 0), 0)
  return base + extra
})

async function addToCart() {
  props.api.toast.success('Товар добавлен в корзину')
}
</script>

<style scoped>
.configurator { display: grid; grid-template-columns: 1fr 1fr; gap: 2rem; padding: 2rem; }
.preview img { width: 100%; border-radius: var(--border-radius); }
.option-group { margin-bottom: 1rem; }
.option-group label { display: block; font-weight: 600; margin-bottom: 0.5rem; }
.option-buttons { display: flex; gap: 0.5rem; flex-wrap: wrap; }
.option-btn {
  padding: 0.5rem 1rem; border: 1px solid var(--card-border);
  border-radius: var(--border-radius); background: var(--color-surface); cursor: pointer;
}
.option-btn.active { background: var(--color-primary); color: #fff; border-color: var(--color-primary); }
.price { font-size: 2rem; font-weight: 700; color: var(--color-primary); margin: 1rem 0; }
.btn { padding: 0.75rem 1.5rem; border: none; border-radius: var(--border-radius); cursor: pointer; font-size: 1rem; }
.btn-primary { background: var(--color-primary); color: #fff; }
</style>
```

---

## 4. Серверная функция — переключение статуса сборки

Вызов серверной функции из кастомного компонента. Серверная функция выполняется в изолированном V8-sandbox и имеет доступ к данным воркспейса через bridge-функции.

### Требования

**Серверная функция** `api/toggle-collected.js` в codespace-репозитории:

```js
// capabilities: query, write
const rec = await getRecord(args.itemId);
const current = rec.requisites[String(args.collectedReqId)] || '0';
const next = current === '1' ? '0' : '1';
await updateRecord(args.itemId, { fields: { [String(args.collectedReqId)]: next } });
return { toggled: next === '1' };
```

### Код компонента

```vue
<script setup>
import { ref } from 'vue'

const props = defineProps({ api: Object })
const loading = ref(false)

async function toggleCollected(itemId) {
  loading.value = true
  try {
    const result = await props.api.callFunction('toggle-collected', {
      itemId,
      collectedReqId: props.api.bindings?.collectedReqId?.id,
    })
    props.api.toast.success(result.toggled ? 'Собрано' : 'Не собрано')
  } catch (e) {
    props.api.toast.error(e.message)
  } finally {
    loading.value = false
  }
}
</script>
```

### Как это работает

1. `api.callFunction('toggle-collected', args)` отправляет `POST /portal/api/fn/portal-components/toggle-collected` с телом `args`.
2. Серверная функция загружает запись через `getRecord`, переключает поле и сохраняет через `updateRecord`.
3. Результат (`{ toggled }`) возвращается в компонент.

---

### Как это работает

1. `api.useRecord(slug)` загружает товар из каталога по slug. Данные товара содержат `_value` (название), `fields.price` (базовая цена), `fields.photo` (фото).
2. `api.useReport(optionsReportId)` загружает таблицу опций. Каждая строка отчёта содержит `name` (название группы, например "Цвет") и `values` (значения через запятую: "Красный, Синий, Зелёный").
3. `api.useReport(modifiersReportId)` загружает ценовые модификаторы. Каждая строка: `option` (группа), `value` (значение), `extra` (наценка в рублях).
4. При клике на кнопку опции `choices[optName]` обновляется реактивно. Computed-свойство `calculatedPrice` автоматически пересчитывает итоговую цену: базовая цена + сумма наценок за выбранные опции.
5. Вся логика ценообразования хранится в данных (отчётах), а не захардкожена в коде компонента.

### Пример конфига модуля

```json
{
  "type": "custom_code",
  "config": {
    "repo": "portal-components",
    "file": "configurator.vue",
    "ref": "main",
    "libraries": [],
    "bindings": {
      "productSlug": "premium-chair",
      "optionsReportId": "report:20",
      "modifiersReportId": "report:21"
    }
  }
}
```
