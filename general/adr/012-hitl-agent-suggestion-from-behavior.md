# ADR-012: HITL-предложение агентов из поведенческих паттернов

**Статус:** accepted

## Контекст

В Integram уже есть три независимых компонента:

| Компонент | Что делает |
|---|---|
| `swarm-memory/behavioral-extractor.js` | LLM извлекает повторяющиеся паттерны из event bus, пишет в swarm memory под `agentId='__behavioral__'` с `MIN_OBSERVATION_N=3`, `MIN_PATTERN_CONF=0.6` |
| `agent-registry` | Хранилище внешних AI-агентов с A2A discovery, HMAC-callback, write-back в EAV |
| `swarm-memory` | Долговременная память агентов, scope=shared, temporal_type=permanent |

Между ними нет моста. Поведенческие паттерны копятся, но никак не превращаются в новых агентов.

Возникает вопрос: должна ли система **сама создавать агентов**, наблюдая за пользователем (по аналогии с Mem0 PersonaMem-v2, который персонализирует поведение существующего агента)?

### Что говорит индустрия

Два раунда web-research (декабрь 2025 / январь 2026) подтвердили:

- **Memory frameworks** (Mem0, Zep, Letta, Cognee) — обогащают существующего агента памятью, не создают новых.
- **Process mining + RPA** (Celonis, UiPath, Tekst) — находят повторяющиеся бизнес-процессы → предлагают RPA-бот. Работают на структурированных event log (SAP, Oracle), не на пользовательских паттернах в LLM-приложении.
- **Auto-design агентов** (ADAS ICLR'25, AutoAgents, EvoAgentX, Voyager, Alita, SkillWeaver) — генерируют агентов/skills из **task description** или **self-experience агента**, не из поведения пользователя.
- **HITL-фреймворки** (Oracle, Microsoft Agent Framework, AWS Bedrock AgentCore) — runtime-конструкции pause/approval/resume для существующих агентов.
- **Survey arXiv 2505.00753v5** ("LLM-Based Human-Agent Collaboration") явно классифицирует *implicit feedback* — наблюдение за действиями пользователя — и констатирует gap: *"current studies are mostly agent-centered... overlooks the potential for agents to proactively monitor and guide human actions"*.

Прямого продукта, который наблюдает поведение пользователя и **предлагает создать агента**, не существует.

Параллельно в 2025 индустрия откатилась от автономных агентов к контролируемым workflow patterns (decodingai, Oct 2025) — рынок готов к "система предлагает → человек подтверждает".

## Рассмотренные альтернативы

### A. Auto-spawn — система сама создаёт агентов

Behavioral-extractor находит паттерн → автоматически создаёт запись в agent-registry → агент начинает работать.

**Проблемы:**
- Privacy: behavioral events могут содержать PII; автосоздание агента увеличивает поверхность утечки.
- False positives: "пользователь часто открывает Inbox" — не повод создавать агента-робота.
- Доверие: пользователь обнаруживает в registry агентов, которых не создавал — теряет контроль.
- Несовместимо с ADR-008 ("HITL обязателен на высоком риске"). Создание агента = высокий риск.

### B. Только manual — игнорировать поведенческие паттерны

Оставить behavioral-extractor как есть, паттерны используются только для personalization внутри уже существующих агентов.

**Проблемы:**
- Теряется ценная сигнатура повторяющейся работы — самый сильный сигнал "это автоматизируемо".
- Пользователь не знает, что он повторяет одно и то же — слепая зона.

### C. HITL-suggestion — система предлагает, пользователь решает ✓ выбрано

Behavioral-extractor продолжает копить паттерны без изменений. Раз в N дней отдельный job:

1. Читает паттерны с `confidence ≥ threshold` и `observation_n ≥ threshold`.
2. Группирует похожие через embeddings (которые уже есть в swarm memory).
3. Для каждой группы генерит **draft agent-card**: название, описание, suggested tools.
4. Показывает draft в постоянном inline-блоке «Предложенные агенты» на странице `/agents/registry` (badge со счётчиком). Notification — только разово при появлении первого suggestion в workspace.
5. На Apply — pre-fills форму создания агента в `agent-registry`; пользователь правит и сохраняет.
6. На Dismiss — паттерн помечается `suggestion_dismissed=true`. Три Dismiss подряд для одного паттерна → `permanently_dismissed`, не предлагать никогда.
7. Третий вариант реакции — Why-not: пользователь пишет короткую причину отказа, она ложится в LLM-промпт следующего расчёта как `negative_example`.

## Решение

### 1. Никогда не создавать агента автоматически

Запись в `_v2_agent_registry` появляется **только** через явное действие пользователя (UI/REST POST `/agents`). Behavioral-job создаёт только **draft в notification**, не запись в registry.

### 2. Suggestion pipeline — отдельный модуль, не часть extractor'а

```
backend/src/api/v2/modules/agent-suggestions/
  router.js
  service.js
  worker.js     — cron, читает swarm memory, генерит draft
```

Behavioral-extractor не знает про agent-registry. Agent-registry не знает про behavioral-memory. Связка живёт в новом модуле — единственная точка, которую можно отключить без побочных эффектов.

### 3. Пороги — стартовые fixed + retirement + план перехода на adaptive

Полный adaptive loop по dismiss-rate (SOC-стандарт 10–30%) рассчитан на поток тысяч сигналов в день. У нас поток ≤1 предложение в неделю на workspace — adaptive по dismiss-rate калибровался бы годами. Решение в два слоя:

**Слой 1 — fixed baseline (MVP):**

```js
SUGGESTION_MIN_OBSERVATIONS = 7    // минимум 7 повторений паттерна
SUGGESTION_MIN_CONFIDENCE   = 0.75 // выше дефолтного 0.6
SUGGESTION_SIMILARITY       = 0.85 // порог cosine для группировки
SUGGESTION_COOLDOWN_DAYS    = 14   // не предлагать повторно раньше
SUGGESTION_PER_RUN_LIMIT    = 1    // одно предложение за прогон
SUGGESTION_INTERVAL_HOURS   = 24   // раз в сутки на workspace
```

**Слой 2 — retirement (быстрый feedback loop без RL):**

Паттерн с 3 подряд Dismiss → `permanently_dismissed`, не предлагается даже после cooldown. Это даёт самокоррекцию без adaptive алгоритма (паттерн Panther: «если правило никогда не даёт TP — retire, не калибруй»).

**Слой 3 — adaptive (Phase 2):** включается только после ≥50 suggestions в телеметрии воркспейса. Цель — Dismiss-rate ∈ [10%, 30%]. Сдвиг порогов automatic по реакции.

Цель — нулевая толерантность к спаму. Лучше упустить кандидата, чем завалить пользователя предложениями. Один спам = доверие потеряно навсегда.

### 4. Privacy-pass обязателен (OWASP LLM02:2025)

Перед формированием draft'а паттерн проходит двойную sanitization (defense in depth):

- Слой 1 — уже в `behavioral-extractor`: `_anonymizePii` (email/phone маскируются).
- Слой 2 — в `agent-suggestions` перед LLM: удаление ID записей (`object#42` → `object`), повторный PII-скан, отбраковка паттернов с PII категории `block` (IBAN, card, SSN).

Если после sanitization паттерн теряет смысл (нет action/description) — отбраковывается. Sanitization выполняется **внутри workspace**, наружу за пределы workspace ничего не уходит.

Базис — OWASP LLM02:2025 «Sensitive Information Disclosure»: «Integrate Data Sanitization Techniques» + «Robust Input Validation».

### 5. Один паттерн ≠ один агент

Job обязан группировать похожие паттерны через embeddings перед генерацией draft'а. Алгоритм группировки — спеко-уровень (greedy cosine threshold для нашего масштаба 50–500 точек на workspace; HDBSCAN для нашего объёма — overkill, на малых корпусах всё классифицируется как noise). Пример: «пользователь меняет статус заказа на 'Доставлен' после звонка клиенту» и «пользователь меняет статус заказа на 'Доставлен' после комментария 'отправлено'» — один draft, не два.

### 6. UX — workspace-уровень, inline-блок, без notifications

Suggestion = ресурс workspace, не персональное сообщение конкретному юзеру. Поэтому:

- **Хранилище:** отдельная таблица `_v2_agent_suggestions` (per workspace). НЕ `_v2_notifications` — те персональны (`username`-таргет), это плохо ложится на «общее для всех админов» предложение.
- **Главный канал (always-on):** persistent блок «Предложенные агенты» на странице `/agents/registry` — все юзеры workspace видят одно и то же. Admin может действовать (Apply/Edit/Dismiss/Why-not), остальные видят кнопки disabled.
- **Badge в сайдбаре:** counter pending-предложений рядом с пунктом «Агенты». Видит каждый.
- **Inline trigger (just-in-time):** при ручном нажатии «Создать агента» — debounced поиск похожих pending-suggestion. Самый продуктивный момент: пользователь уже хочет агента.
- **Никаких персональных notifications.** Notification fatigue — лечится тем, что мы их вообще не шлём.

Apply — атомарный CAS (`UPDATE ... WHERE status='pending' RETURNING id`): если два admin'а одновременно нажмут Apply, агент создастся один раз, второй получит «уже обработано».

Принципы — Agentic UX «Invisible, not absent» (Medium, 2025): agent живёт внутри существующего workflow, не в отдельной странице. Grammarly inline + Gmail Smart Compose — эталоны.

### 7. Why-not вместо плоского Dismiss

Третий вариант реакции (кроме Apply/Dismiss): пользователь пишет короткую причину отказа (1–2 предложения). Причина пишется в `agent_memory` под `agent_id='__behavioral__'` с тегом `rejection_reason` и идёт в LLM-промпт следующего расчёта как `negative_example`. Это даёт closed-loop learning без RL-инфраструктуры — LLM «учится» какие группы паттернов пользователь систематически отвергает.

### 8. Telemetry — обязательна

Каждое предложение логируется: какие паттерны попали, какой draft сгенерирован, реакция пользователя (Apply / Edit / Dismiss / Why-not / ignored), latency реакции. Без этого нельзя ни оценить ценность фичи, ни включить Phase 2 adaptive thresholds.

## Последствия

**Запрещено:**
- Создавать запись в `agent-registry` из любого автоматического процесса.
- Предлагать агента на основании одного паттерна без группировки.
- Передавать в LLM (для генерации draft'а) сырые behavioral events без sanitization.
- Понижать пороги без анализа Apply/Dismiss-телеметрии за предыдущий период.

**Упрощается:**
- Пользователь видит свои повторяющиеся действия — слепая зона исчезает.
- Behavioral-memory получает осмысленное применение (раньше только personalization).
- Agent-registry остаётся чистым: каждый агент имеет явного автора-пользователя.

**Усложняется:**
- Новый cron-job + worker, требует мониторинга.
- Калибровка порогов: первые недели — high false-positive rate, нужно сдвигать вверх по реакции.
- Sanitization-pass — нетривиальный, требует тестов на реальных паттернах.
- Three-channel UX (inline / notification / just-in-time) требует синхронизации состояний.
- Why-not flow требует UI-поле для свободного текста и LLM-промпт-инженерии для использования causes в следующих расчётах.

## Антипаттерны — явные нарушения этого ADR

| Антипаттерн | Почему неправильно |
|---|---|
| «Сделаем флаг `auto_create_agent=true` для опытных пользователей» | Auto-spawn запрещён без исключений. Любой опытный пользователь предпочтёт нажать Apply один раз, чем разбираться с непрошенными агентами в registry. |
| «Будем предлагать после каждого нового паттерна» | Без cooldown пользователь утонет в notifications. Минимум 14 дней между предложениями для одного паттерна. |
| «Сырые events в LLM — он сам выберет нужное» | PII попадает в LLM-провайдера. Sanitization до LLM — обязательна. |
| «Один паттерн = один draft, группировку добавим потом» | Спам с первого же запуска. Группировка — часть MVP, не оптимизация. |
| «Suggestion = новая таблица с FK на agents» | Усложняет схему без причины. Notifications уже умеют payload и lifecycle. |
| «Отключить, если пользователь жмёт Dismiss» | Один Dismiss = «не этот паттерн», не «никогда». Permanently disable — только после 3 Dismiss подряд (retirement). Глобальный opt-out — отдельная настройка workspace. |
| «Fixed thresholds навсегда — раз работает, не трогаем» | Production-ML стандарт (Aerospike, Buzzclan, SEI/CMU Augur): пороги дрейфуют, нужен план перехода на adaptive после сбора телеметрии. Иначе либо тишина, либо спам с течением времени. |
| «Notification на каждое предложение — пользователь должен знать» | Smashing Magazine 2025: «Mute by default = провал». Notification только разово, основной канал — inline-блок. |
| «HDBSCAN потому что это SOTA» | На малых корпусах (≤500 точек) HDBSCAN классифицирует всё как noise, требует UMAP + hyperparameter tuning. Greedy cosine threshold проще и адекватнее. SOTA ≠ optimal для нашего масштаба. |
| «Apply / Dismiss — простая бинарная реакция, Why-not переусложнение» | Бинарная реакция теряет сигнал. Why-not даёт LLM negative examples для следующих расчётов — closed-loop learning без RL. |

## Источники

- Survey "LLM-Based Human-Agent Collaboration and Interaction Systems" (arXiv 2505.00753v5, май 2025) — implicit feedback classification, agent-centered gap.
- Anthropic "Building effective agents" (декабрь 2024) — HITL на высокорисковых действиях.
- Mem0, Zep, Letta — memory frameworks (анализ ниши, январь 2026).
- Celonis / UiPath process mining + RPA suggestions — соседняя ниша на структурированных процессах.
- OWASP "LLM02:2025 Sensitive Information Disclosure" (genai.owasp.org) — формальный baseline для sanitization PII перед LLM.
- "Agentic UX: 7 principles for designing systems with agents" (Medium, 2025) — Invisible-not-absent, Grammarly inline pattern, attention-appropriate notifications.
- Conifers AI / Panther / arXiv 2605.08316v1 "AI-Driven Security Alert Screening" (2025) — целевой FP-rate 10–30%, retirement правил, adaptive recalibration loop.
- SEI/CMU Augur (2022) + Aerospike model drift guide — fixed thresholds в production-ML как антипаттерн.
- TDS "Clustering sentence embeddings to identify intents in short text" — HDBSCAN на малых корпусах требует UMAP + tuning, на маленьких N плохо.
- ADR-008 (мультиагентная оркестрация) — HITL на высоком риске.
- ADR-005 (event bus) — источник events для behavioral-extractor.
