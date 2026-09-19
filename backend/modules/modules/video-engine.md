# Video Engine Module

Конвейер обучающих видео: сценарий (JSON: шаги + текст озвучки единым документом) → запись экрана headless chromium (Playwright screencast) → TTS (edge-tts) → монтаж (ffmpeg) → mp4 в файловом дереве воркспейса.

## Architecture

| Файл | Роль |
|---|---|
| `scenario.js` | Валидация и нормализация сценария (`validateScenario`), лимиты |
| `service.js` | Джобы: создание/статус/список/отмена; хранение в `public.agent_memory` |
| `router.js` | REST `/api/v2/:db/video/*` |
| `worker.js` | BullMQ-воркер (`initVideoEngineWorker`) + процессор `processVideoJob` (экспортирован для inline-fallback) |
| `pipeline.js` | Стадии `tts → recording → rendering → done`; ошибка любой стадии — `stage:'error'` + проброс |
| `tts/provider.js` | Выбор провайдера env'ом `VIDEO_TTS_PROVIDER` (`edge` \| `rhvoice`), общий интерфейс `synthesizeScenario({dir, steps}) → [{file, durationMs, stepIndex}]` |
| `tts/edge-tts.js` | edge-tts CLI: mp3 + SRT с word boundaries; длительность фразы = конец последнего слова |
| `tts/rhvoice.js` | Офлайн-фолбэк — заглушка: выбор `VIDEO_TTS_PROVIDER=rhvoice` падает громко, не уводит сегменты в тишину |
| `recorder.js` | Исполнение сценария в headless chromium, screencast `raw.webm`, курсор/подсветка кликов, гейты ожидания, pacing по фразам, кадры-сториборд |
| `assembly.js` | Монтаж: план выхода (фриз-хвосты, сжатие долгих wait) → concat → аудио-граф (adelay фраз по выходным оффсетам) → мукс h264+aac |
| `paths.js` | Каталог артефактов джобы: `UPLOAD_DIR/<db>/video-engine/<jobId>` (корень тот же, что у files) |

## Двухпроходность

1. **TTS до записи**: каждая фраза синтезируется раньше, длительность берётся из SRT (word timings).
2. **Запись с pacing**: шаг держится не короче фразы + 800 мс хвоста — фриз-хвосты сборке почти не нужны.
3. **Монтаж по факту**: сборка работает по фактическим `marks` записи (`{stepIndex, startMs, actualMs, idle}`), а не по плану-оценке (`buildRunPlan` — чистая функция-оценка).

## Опции сценария

| Опция | Env | Default |
|---|---|---|
| База для `navigate` | `VIDEO_BASE_URL` | `http://localhost:8081` |
| Голос TTS | `VIDEO_TTS_VOICE` | `ru-RU-DmitryNeural` |
| Бинарник TTS | `VIDEO_TTS_BIN` | `edge-tts` |
| Провайдер TTS | `VIDEO_TTS_PROVIDER` | `edge` (`rhvoice` — не реализован, падает) |
| Держать idle не длиннее | `VIDEO_MAX_IDLE_KEEP_MS` | `3000` |
| Целевая длина сжатого wait | `VIDEO_MIN_IDLE_MS` | `1500` |
| Хвост после фразы при фризе | `VIDEO_NARRATION_PAD_MS` | `800` |
| Доводы запуска chromium (пробел-разделитель) | `VIDEO_RECORDER_CHROMIUM_ARGS` | пусто; на машинах с GPU-крашами — `--disable-gpu --no-zygote` |
| Корень артефактов (как у files) | `UPLOAD_DIR` | `integram-server/download` |

## Формат сценария

```json
{
  "title": "Создание клиента",
  "viewport": { "width": 1600, "height": 900 },
  "steps": [
    { "type": "navigate", "url": "/myws", "settle": 3000,
      "narration": "Открываем воркспейс" },
    { "type": "click", "target": { "role": "button", "name": "Создать" } },
    { "type": "type", "target": { "placeholder": "Имя" }, "text": "Иван",
      "narration": "Вводим имя клиента" },
    { "type": "wait", "gate": { "selector": ".done", "timeoutMs": 30000 } },
    { "type": "shot", "label": "result" }
  ]
}
```

- **Типы шагов**: `navigate`, `click`, `type`, `press`, `scroll`, `wait`, `shot`.
- **target** (для `click`/`type`; для `press` опционален — без него клавиша жмётся на сфокусированном): `{ role | placeholder | text | selector }`.
- **`wait`+gate**: `selector` (появление), `selectorGone` (исчезновение), `textStableMs` (текст страницы стабилен), `timeoutMs` (1 000–600 000, default 120 000).
- **narration** на любом шаге — текст озвучки; на экране синхронно показывается глава (`page.screencast.showChapter`, первые 80 символов).
- **Лимиты**: ≤ 200 шагов; narration ≤ 2 000 символов; viewport 640–1920 × 400–1080 (default 1600×900); `navigate.url` — только относительный путь (`/...`, разрешается против `VIDEO_BASE_URL`); `settle` ≤ 60 000 мс.

## REST

Монтируется как `router.use('/:db/video', requireModule('video'), videoRouter)` — модуль opt-in.

- `POST /jobs` — создать джобу (тело — сценарий); постановка в очередь BullMQ `video-engine` с `attempts: 1` (полный перерендер дорог — повтор решает вызывающий); при недоступном Redis — inline-прогон воркера через `sideEffect`
- `POST /screen` — загрузка записи с экрана: multipart `file` (webm/mp4, ≤500MB) + `title`. Джоба `kind=screen`: только монтаж (normalize fps=30 в размере исходника + аудио из исходника), без TTS/записи
- `GET /jobs` — список (последние 100 статусов джоб воркспейса)
- `GET /jobs/:id` — статус: `{ jobId, meta, status }`; `status.stage` — `queued | tts | recording | rendering | done | error | cancelled`, `progress` 0–100, `resultFile` при готовности
- `POST /jobs/:id/cancel` — атомарная отмена одним UPDATE (`WHERE stage NOT IN ('done','error','cancelled')`); false — джобы нет или она уже терминальна
- `GET /jobs/:id/download` — mp4; 409 `NOT_READY` без результата
- `GET /jobs/:id/storyboard` — кадры `frame-NNN.jpg` с подписями narration; читается ДО готовности mp4 — сломанный шаг виден, джобу можно отменить даром
- `GET /jobs/:id/storyboard/:name` — кадр (имя строго `frame-\d{3}.jpg`)

## Воркер

- Очередь BullMQ `video-engine`; `concurrency: 2`, `lockDuration: 900_000` (запись живёт десятки минут — как normalizer-extract, иначе BullMQ отдаст джобу второму воркеру); **`attempts: 1` — без автоповторов**.
- Воркер стартует в отдельном процессе `backend/scripts/start-worker.js` (`initVideoEngineWorker`); без Redis воркер не поднимается — REST и инструменты сами гонят джобу in-process.
- По завершении — уведомление («Видео готово» / «Ошибка рендера видео»), тип `system`.

## Монтаж (`assembly.js`)

- **Выходной план** (`buildOutputPlan`): если фраза длиннее записанного сегмента — фриз-хвост последнего кадра на `phraseMs + narrationPadMs`; долгий `wait` (idle > `maxIdleKeepMs`) сжимается до `minIdleMs` фриз-кадром.
- Посегментное кодирование (`libx264, crf 23, 30 fps`, масштаб+pad под viewport) → concat demuxer → `normalized.mp4`.
- **Аудио-граф** (`buildAudioGraph`): тишина-база `anullsrc` 48 кГц + `adelay` каждой фразы по ВЫХОДНОМУ оффсету шага; `aresample=48000` перед `amix` обязателен (mp3 edge-tts — 24 кГц, без выравнивания — рассинхрон).
- Мукс: видеопоток копируется без перекодирования, аудио — aac; без фраз — `-an`.

## Хранение

Джобы живут в `public.agent_memory` с `agent_id 'video-engine'`, ключи `job:<id>:meta|scenario|status` (прямой UPSERT, паттерн normalizer — минуя swarm-memory консолидацию, которая редиректит похожие ключи между джобами). Артефакты — на диске в `UPLOAD_DIR/<db>/video-engine/<jobId>/`: `raw.webm`, `seg-<i>.mp3/.srt`, `frame-NNN.jpg`, `out-<i>.mp4`, `result.mp4`.

## Фронтенд

Страница `/:db/video` (`frontend/src/views/video/VideoPage.vue`, маршрут `video`): список джоб с прогрессом по стадиям, предпросмотр сториборда, встроенный плеер (blob + revoke), отмена. Сервис — `frontend/src/services/video.js`.

- Создание джобы — кнопка «Новый ролик» на странице: форма шагов с озвучкой
  (`components/video/ScenarioEditor.vue`), клиентская валидация зеркалит серверную
  (`utils/videoScenario.js` — сервер остаётся судьёй), предпросмотр-таймлайн с оценкой
  длительности (эвристика, факт считает рекордер по меткам). Либо REST / vid_create_job.
  Справка по разделу — HelpDialog, топик `video`.

Два режима создания — SplitButton «Новый ролик»: по сценарию (форма редактора)
или с моего экрана (`getDisplayMedia` + `MediaRecorder` в браузере пользователя:
системный звук, опционально микрофон; webm/mp4 загружается и конвертируется в
mp4 на сервере). Отмена screen-джобы действует до завершения обработки.

## MCP-инструменты (группа `video`, gated)

| Инструмент | Tier | Описание |
|---|---|---|
| `vid_create_job` | MEDIUM | Создать джобу (сценарий) — та же постановка в очередь, что и REST |
| `vid_get_job` | LOW | Статус джобы |
| `vid_list_jobs` | LOW | Список джоб воркспейса |
| `vid_cancel_job` | MEDIUM | Отменить (not_cancellable, если терминальна) |
| `vid_validate_scenario` | LOW | Проверить сценарий по схеме, ничего не создавая |

Заслон: `module-gate.js` — группа `video`, `defaultEnabled: false`, префикс `vid_`; инструменты скрыты из `GET /ai/tools` в воркспейсах без `settings.modules.video = true`. Сториборд в MCP не отдаётся — изображения непредставимы в JSON, кадры доступны только по REST.

## Access control

- **REST** — `requireModule('video')` на маунте: выключенный модуль отвечает 403 `MODULE_DISABLED`.
- **Включение**: `modules.video = true` в настройках воркспейса (`DEFAULT_MODULES` — `video: false`).
- **Tools** — `module-gate.js` скрывает `vid_*` и отказывает в `permissionMiddleware` для выключенного модуля.

## Известные ограничения

- **edge-tts — неофициальный API Microsoft** (Edge Read Aloud), может отвалиться; выбор провайдера за интерфейсом (`tts/provider.js`), офлайн-фолбэк RhVoice объявлен, но не реализован — попытка включения падает громко, а не пишет тишину.
- **Отмена действует на границе стадий** конвейера (`tts → recording → rendering`); внутри записи шаг не прерывается.
- Зависший edge-tts убивается SIGKILL через 2 мин (timeout у spawn).
- Список джоб — последние 100 (`LIMIT 100` по `agent_memory`); история джоб не чистится автоматически.
- Для прод-деплоя нужны: chromium (`npx playwright install chromium`), edge-tts CLI (pipx/uvx), ffmpeg.

## Третий вызывающий постановки

Пути постановки джобы: REST (этот роутер), AI-инструмент vid_create_job, автоматизация `run_video_job` (modules/automations/run-video-job.js) — у всех одна createJob + очередь 'video-engine', один in-process fallback при недоступном Redis. Конвейер и его лимиты существуют в одном экземпляре — в этом модуле.
