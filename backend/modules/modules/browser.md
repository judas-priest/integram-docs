# Browser Service

Отдельный сервис для веб-автоматизации через управляемый браузер. Живёт в `browser/`, запускается как отдельный процесс на порту 3099.

Не только парсинг данных — универсальный инструмент для любых задач с браузером: заполнение форм, авторизация, скриншоты, перехват сетевых запросов.

## Интеграция

### run_script bridge

В `isolated-runner.js` доступна bridge-функция `browse(query, source?)`:

```js
// Внутри run_script:
const results = await browse('кабель hdmi 2м', 'wildberries');
// → [{name, price, url, source}, ...]
const minPrice = Math.min(...results.map(r => r.price).filter(Boolean));
output(minPrice);
```

Использует rate limit `fetch` (50 вызовов на исполнение). Под капотом делает HTTP-запрос к `http://localhost:3099/search`.

### AI tool

`search_prices` — TIER_LOW, доступен через MCP и in-app агента.

```
search_prices(query="кабель hdmi 2м", source="wildberries", limit=5)
→ { items: [...], query, source, total }
```

### Script Button

Из браузерного скрипта через fetch-proxy:

```js
const r = await fetch('http://localhost:3099/search?q=' + encodeURIComponent(row['Товар']) + '&source=wildberries');
const items = JSON.parse(r.body);
const min = Math.min(...items.map(i => i.price).filter(Boolean));
output(min);
```

## Источники

| Source | Файл | Метод |
|---|---|---|
| komus       | `browser/scrapers/komus.js`       | Playwright stealth + DOM parsing |
| wildberries | `browser/scrapers/wildberries.js` | Playwright stealth + DOM parsing |
| samson      | `browser/scrapers/samson.js`      | Playwright stealth + DOM parsing |

## Concurrency Pool

`browser/pool.js` — семафор, ограничивает одновременные browser сессии (default: 2, env `BROWSER_MAX_CONCURRENT`). Каждый Chromium = 200-500MB RAM. При массовом импорте (38 автоматизаций одновременно) без pool сервер падает с OOM. С pool — запросы встают в очередь, обрабатываются по 2.

`GET /health` возвращает `{ status: "ok", pool: { running, queued, max }, chromiumOk, cacheSize }`.

## Architecture

| File | Purpose |
|---|---|
| `index.js` | Express server, `/search` + `/health`, cache, domain delay, graceful shutdown |
| `pool.js` | Concurrency semaphore (max 2), slot timeout (90s), queue cap (20), queue wait timeout (60s) |
| `scraper-base.js` | Shared browser setup: launch, context, UA rotation, resource blocking, cleanup |
| `ua-list.js` | Recent Chrome user-agent strings for rotation |
| `scrapers/*.js` | Per-source scraping logic (evaluate only) |

## Добавление источника

1. Create `browser/scrapers/<name>.js`
2. Import `createScraperPage` from `../scraper-base.js`
3. Choose browser engine: `playwright-extra` with StealthPlugin (most sources) or `rebrowser-playwright` (for sites with advanced bot detection — pass `{ useExecPath: true }` to `createScraperPage`)
4. Export `search(query, { limit })` — use `createScraperPage(chromium)` for setup, write only the `page.evaluate` logic
5. Add source name to `supportedSources` in `index.js`
