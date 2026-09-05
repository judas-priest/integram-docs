# Module: automations

**Path:** `src/api/v2/modules/automations/`
**Files:** `router.js`, `service.js`, `worker.js`, `listeners.js`, `isolated-runner.js`, `dispatchers.js`, `execution.js`, `ai-analysis.js`, `scheduler.js`, `batch.js`, `lifecycle.js`, `telegram-helpers.js`, `helpers.js`, `pm-triggers.js`, `run-server-function.js`, `update-related.js` (`ls` of the module directory, minus `__tests__/`)
**Base URL:** `/api/v2/:db/automations/...`
**Auth:** JWT required. List/get: any authenticated user. Create/update/delete: `admin`. Manual run: `editor`.

## Purpose

Event-driven automation rules. Each automation has a trigger, optional condition, and one or more actions. Triggered by EAV object events, cron schedules, webhooks, form submissions, or AI analysis insights.

## Endpoints

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/automations` | any | List automations for workspace (optional `?botId=N` to filter by Telegram bot) |
| POST | `/automations` | admin | Create automation (idempotency key supported) |
| GET | `/automations/:id` | any | Get automation details |
| PATCH | `/automations/:id` | admin | Update automation |
| DELETE | `/automations/:id` | admin | Delete automation |
| POST | `/automations/:id/run` | editor | Manually trigger automation with optional context |
| GET | `/automations/:id/runs` | any | Execution history (limit/offset) |
| GET | `/automations/insights` | editor | AI analysis insights for workspace |
| POST | `/automations/insights/dismiss` | editor | Dismiss an insight |
| POST | `/automations/insights/feedback` | editor | Rate insight quality (+1/-1) |
| POST | `/automations/:id/run-batch` | editor | Start batch automation run (returns batchId) |
| GET | `/automations/batch-status/:batchId` | viewer | Poll batch progress (returns done, total, complete, error) |
| POST | `/automations/seed-system` | admin | Restore missing system automations (idempotent) |

## Create Body (`POST /automations`)

| Field | Required | Description |
|-------|----------|-------------|
| `name` | yes | 1–255 characters |
| `trigger` | yes | `{ type, … }` — `type` is one of the [trigger types](#trigger-types) accepted by the router's Zod schema; the rest of the object is trigger-specific |
| `condition` | no | Condition object or `null`. Omitted ⇒ `null` — see [Condition](#condition) |
| `actions` | yes | Non-empty array of `{ type, … }` — see [Action Types](#action-types) |
| `active` | no | `false` creates the automation switched off. Omitted ⇒ enabled |

Unknown keys are stripped by `validate()` before the body reaches the service, so a
field this table does not list is silently dropped rather than rejected.

**`active: false` — created but not running.** The automation is stored with
`active = FALSE` and never starts on its own:

- it does not enter the schedule — `onAutomationSaved()` (`worker.js:369-388`) reads
  `active` off the `automation.saved` event and, when it is false, *removes* the
  BullMQ repeatable job key instead of upserting it (removal, not just omission, so
  that switching an existing automation off also takes it out of the schedule);
- it does not fire on events — every dispatcher in `dispatchers.js` selects
  `WHERE active = TRUE`, and `syncCronJobs()` reads the same condition on startup.

`PATCH /automations/:id` with `{ "active": true }` enables it; the same
`automation.saved` event then registers the cron job.

Omitting `active` creates an **enabled** automation — the long-standing default, kept
unchanged (`active = true` in `createAutomation`, column is `NOT NULL DEFAULT TRUE`).

**AI/MCP tool:** `create_automation` takes `active` as well, with the same meaning and
the same default. The value survives the HITL confirmation round-trip — it is part of
the pending payload, not re-read from the model's original call.

## Trigger Types

| Type | Description |
|------|-------------|
| `on_create` | Object created in a table (`typeId`) |
| `on_update` | Object updated in a table (`typeId`). Optional `condReqId`: when set, fires only if that specific reqId was among the changed fields. |
| `on_delete` | Object deleted from a table |
| `schedule` | Cron expression (`cron` field, e.g. `0 9 * * 1`) |
| `manual` | Only via `POST /automations/:id/run` |
| `on_deadline` | Object's date field reaches threshold (`dateReqId`, `offsetMinutes`) |
| `on_webhook` | Incoming webhook with matching `token` |
| `on_form_submit` | Form submission |
| `ai_analysis` | Periodic LLM analysis of selected tables (`typeIds`, `prompt`) |
| `on_metric_threshold` | Timeseries metric crosses threshold |
| `on_metric_silence` | Timeseries metric stops updating |
| `on_document` | Document created, updated, or status changed |
| `on_telegram_command` | Telegram bot command received. `trigger.command` field matches the `automationTrigger` from bot config. Optional `botId` field to filter by specific bot. Template vars: `_command`, `_args`, `_chat_id`, `_from_username`, `_from_first_name` |
| `on_telegram_message` | Non-command Telegram message — text **or** media. Fields: `pattern` (comma-separated keywords), `matchMode` ('contains'\|'exact'), `messageTypes` (array, intake rules only), `botId` (optional). See below for matching rules and template vars. |
| `on_ncl_run_completed` | Nightcall run finished (bus event `ncl.run.completed`). Payload: `runId`, `verdict`, `rationale`. |
| `on_ncl_requirement_created` | Nightcall requirement created (`ncl.requirement.created`). Payload: `requirementId`, `stableId`, `modality`, `status`, `rawText`. |
| `on_ncl_evidence_stale` | Nightcall evidence went stale (`ncl.evidence.stale`). Payload: `updated`, `reason`, `requirementId`, `sourceId`. |
| `on_issue_created` | PM issue created. Payload: `issueId`, `number`, `type`, `assigneeId`, `reporterId`. |
| `on_issue_status_changed` | PM issue moved between statuses. Payload: `issueId`, `number`, `from`, `to`, `actorId`. |
| `on_sprint_started` | PM sprint started. Payload: `sprintId`, `name`. |
| `on_sprint_completed` | PM sprint completed. Payload: `sprintId`, `name`, `stats`. |
| `pm_deadline` | Daily scan of open PM issues whose `due_date` falls within `config.days` (default 1). Runs as a BullMQ repeatable job at `config.hour` UTC (default 7); DB-fallback scheduler sweeps it too. Dedup via `_v2_pm_deadline_log` — one fire per issue per day. Template vars: `{{pm_issue_id}}`, `{{pm_number}}`, `{{pm_title}}`, `{{pm_status}}`, `{{pm_due_date}}`, `{{pm_assignee_id}}`, `{{pm_assignee_email}}`, `{{pm_assignee_username}}`. |
| `on_issue_commented` | Comment added to a PM issue (bus `pm.issue.commented`). Payload via `ctx.pm`: `issueId`, `number`, `commentId`, `authorId`, `mentions`, plus `pm_title`/`pm_status`/`pm_due_date`/`pm_assignee_id`. |
| `on_issue_updated` | PM issue fields changed (bus `pm.issue.updated`). Payload via `ctx.pm`: `issueId`, `number`, `fields`, `actorId`, plus the same `pm_*` context. |

The three `on_ncl_*` types are dispatched from `automations/listeners.js` off the
`ncl.*` bus events; the Nightcall module is opt-in, so in workspaces with it off
they simply never fire. The four PM types go through `dispatchPmTrigger`
(`automations/pm-triggers.js`), called from `pm/listeners.js` — that indirection
never throws, because a failing trigger must not break the creation of an issue.

### `on_telegram_message` — keyword rules vs intake rules

`pattern` decides which of the two a rule is:

| | keyword rule (`pattern` set) | intake rule (`pattern` empty/absent) |
|---|---|---|
| Matches | messages containing/equal to a keyword | **every** message from the bot |
| Chain | first match wins — later keyword rules are skipped | always runs; does not consume the keyword chain |
| Narrowing | `matchMode` | `messageTypes: ["voice","document"]` |

The two coexist: an intake rule that logs every message into a table runs
alongside a keyword rule that replies to `привет`, on the same bot.

Media messages (voice, audio, video, video_note, animation, document, photo,
sticker) and service messages reach the trigger too. For media, `pattern` is
matched against the caption, and `messageType` names the kind. Files themselves
stay in Telegram — `_file_id` is what `getFile` needs to fetch them.

**Template vars** (`buildEnv`):

| Var | Meaning |
|---|---|
| `_message`, `_tgMessage` | text, or the caption for captioned media |
| `_caption` | caption only |
| `_message_type` | `text`, `voice`, `document`, `photo`, `service`, … |
| `_message_id`, `_date`, `_reply_to_message_id` | message identity; `_date` is ISO 8601 |
| `_chat_id`, `_chat_title`, `_chat_type` | chat identity |
| `_from_user_id`, `_from_username`, `_from_first_name`, `_from_last_name`, `_from_is_bot` | author; `_from_user_id` is the only stable key — usernames change |
| `_file_id`, `_file_unique_id`, `_file_name`, `_file_size`, `_mime_type`, `_duration` | attachment, empty for text |
| `_edited` | `1` when the update is an edit of an earlier message |

Example — log every group message into a table:

```json
{
  "trigger": { "type": "on_telegram_message", "botId": 7 },
  "actions": [{
    "type": "create_object",
    "typeId": 364,
    "value": "{{_from_first_name}}: {{_message_type}}",
    "requisites": {
      "365": "{{_date}}", "368": "{{_from_user_id}}",
      "370": "{{_message}}", "371": "{{_message_type}}",
      "372": "{{_message_id}}", "797": "{{_file_name}}"
    }
  }]
}
```

> **Note — triggers only available via MCP/tool (not REST):** the following trigger types exist in `dispatchers.js` but are absent from `TRIGGER_TYPES` in the router's Zod validation schema, so they cannot be created or updated via REST API. They work only when written directly via MCP tool or SQL: `on_telegram_pre_checkout`, `on_telegram_shipping`, `on_telegram_payment`, `on_telegram_inline`, `on_telegram_join_request`, `on_telegram_business_connection`, `on_telegram_business_message`, `on_bot_chat_member`.
>
> The Nightcall and PM types are **not** on that list — `on_ncl_run_completed`, `on_ncl_requirement_created`, `on_ncl_evidence_stale`, `on_issue_created`, `on_issue_status_changed`, `on_sprint_started`, `on_sprint_completed`, `pm_deadline`, `on_issue_commented` and `on_issue_updated` are all in `TRIGGER_TYPES` and are accepted over REST like any other. The list above is the exception set, not a summary of what is new.

## WebSocket Events

Automation lifecycle broadcasts to connected clients via `automations` WS channel:

| Event | When | Data |
|-------|------|------|
| `automation.started` | Condition passed, before actions execute | `{ automationId, objId, typeId }` |
| `automation.finished` | All actions complete (ok/error/suspended) | `{ automationId, objId, typeId, status }` |

Frontend uses these for real-time feedback: toast on start/finish, pulsing badge during execution.

## Action Types

`update_field`, `set_requisite`, `set_field` (alias for `set_requisite`), `update_related`, `update_related_records`, `create_object`, `delete_object`, `send_notification`, `send_notification_to_group`, `fire_webhook`, `http_request`, `run_connector`, `send_email`, `send_sms`, `send_telegram`, `delete_telegram_message`, `send_chat_action`, `run_ai_agent`, `run_script`, `run_server_function`, `request_approval`, `if_else`, `switch`, `transform`, `wait_delay`, `delegate_to_agent`, `invoke_agent`, `create_cdek_shipment`, `reconcile_cdek`, `create_teamchat_room`, `telegram_forward`, `send_invoice`, `answer_inline_query`, `telegram_ban`, `telegram_unban`, `telegram_restrict`, `telegram_promote`, `telegram_approve_join`, `telegram_decline_join`, `telegram_pin`, `telegram_unpin`, `telegram_get_chat`, `telegram_download_file`, `telegram_post_story`, `answer_shipping`, `telegram_business_reply`, `create_issue`, `update_issue`

### create_object

Creates a new EAV object. Supports template variables in all fields including `parentId`.

```json
{
  "type": "create_object",
  "typeId": 151435,
  "parentId": "{{id}}",
  "value": "{{req_328}}",
  "requisites": {
    "151436": "{{NOW}}",
    "151437": "{{req_328}}"
  }
}
```

- `typeId` — target table type ID
- `parentId` — parent object ID. Supports template variables (e.g. `"{{id}}"` for trigger object). Defaults to 1 (workspace root) if omitted.
- `value` — display name for the new record. Supports template variables.
- `requisites` — map of `reqId → value`. Values support template variables.

After the action, `{{_created_id}}` holds the new record's id — used by later actions
to attach files or update the record just created.

### create_issue

Creates a PM issue. PM issues live in satellite tables, so `create_object` cannot
reach them — this action goes through `pm/service.js`.

```json
{
  "type": "create_issue",
  "title": "Follow-up for {{pm_number}}",
  "issue_type": "task",
  "priority": "high",
  "assignee_id": "{{pm_assignee_id}}",
  "due_date": "{{NOW}}",
  "labels": "support, urgent"
}
```

- `type` is the action discriminator; the issue type is passed as `issue_type` (default `task`)
- other fields (all optional): `description`, `priority` (default `medium`), `assignee_id`, `parent_id`, `board_id`, `sprint_id`, `due_date`, `estimate`, `labels` (comma-separated string)
- outputs: `{{_created_issue_id}}`, `{{_created_issue_number}}`

### update_issue

Updates an existing PM issue.

```json
{ "type": "update_issue", "id": "{{pm_issue_id}}", "status": "done" }
```

- `id` — required; issue to update (template vars allowed, e.g. `{{pm_issue_id}}` from a PM trigger)
- optional fields: `title`, `description`, `status`, `priority`, `assignee_id`, `sprint_id`, `parent_id`, `estimate`, `due_date`, `labels`, `issue_type`

### telegram_download_file

Pulls media a user sent to the bot into workspace files, so it enters the normal
processing pipeline — audio is transcribed, scans go through OCR, everything becomes
searchable. Without it an `on_telegram_message` rule can record that a voice note
arrived, but never its contents: the file stays on Telegram's servers.

```json
{ "type": "telegram_download_file", "outputReqId": 797 }
```

- `fileId` — file to fetch; defaults to `{{_file_id}}` of the triggering message
- `botId` / `botToken` — defaults to the bot that received the message
- `filename` — stored name; defaults to `{{_file_name}}`, then to the tail of Telegram's
  `file_path` (voice notes arrive nameless, and the `.oga` extension has to survive)
- `objectId` — record to attach the file to; defaults to `{{_created_id}}` of a preceding
  `create_object`, then to the triggering object
- `subdir` — target subdirectory in workspace files
- `outputReqId` — field to write the stored filename into

Sets `_downloaded_file_id`, `_downloaded_filename`, `_downloaded_size` and
`_downloaded_status` for later actions. A failure sets `_download_error` and lets the
run continue. Bot API caps downloads at **20 MB** — larger media cannot be fetched by
any bot and is reported through `_download_error`.

### update_related

Updates a field on **all child records** of the triggering object.

```json
{ "type": "update_related", "childTypeId": 315, "reqId": 359, "value": "true" }
{ "type": "update_related", "childTypeId": 315, "reqId": 362, "delta": -1 }
```

- `childTypeId` — type ID of the child table (must have `up = objId`)
- `reqId` — field to update in each child record
- `value` — new value; if text and field is a ref column, resolved to record ID by name
- `delta` — numeric increment/decrement (mutually exclusive with `value`)

### send_telegram

Sends a Telegram message via Bot API. Supports two token sources:

**Manual (hardcoded token):**
```json
{
  "type": "send_telegram",
  "integrationMode": "manual",
  "botToken": "7812345678:AAH...",
  "chatId": "-1001234567890",
  "message": "Заказ №{{val}} оформлен. Сумма: {{req_362}}"
}
```

**From credentials table (ARCH-2):**
```json
{
  "type": "send_telegram",
  "integrationMode": "table",
  "integrationId": 42,
  "tokenReqId": 851,
  "chatId": "-1001234567890",
  "message": "Заказ №{{val}} оформлен."
}
```

- `integrationMode` — `"manual"` (default) or `"table"`. If `"table"`: reads the bot token from EAV field `tokenReqId` of object `integrationId`.
- `message` — supports `{{val}}` (trigger object name) and `{{req_NNN}}` (field values) template variables.
- `parseMode` — optional. Telegram parse mode: `"HTML"` or `"Markdown"`. Omit for plain text.
- `buttonText` — optional. Text displayed on an inline keyboard button appended to the message.
- `buttonReqId` — optional. reqId of the field to update on the trigger object when the button is clicked.
- `buttonValue` — optional. Value to write to `buttonReqId` when the button is clicked.
- Button callback data format: `u:<objId>:<reqId>:<value>` — handled by the bot's callback query webhook.
- Shared helper: `backend/src/api/v2/utils/telegram.js` → `telegramSendMessage()`.

**Multi-recipient broadcast (`recipients.fromTable`):**

Instead of a single `chatId`, send the same message to every active member of an "employees-like" table:

```json
{
  "type": "send_telegram",
  "integrationMode": "table",
  "integrationId": 151433,
  "tokenReqId": 151432,
  "recipients": {
    "fromTable": {
      "typeId": 313,
      "chatIdReqId": 354,
      "activeReqId": 356,
      "roleReqId": 380,
      "roleIds": [419, 420]
    }
  },
  "message": "Новый заказ №{{val}}, сумма {{req_340}} руб."
}
```

- `typeId` — table to read members from (e.g. `Сотрудники`)
- `chatIdReqId` — column with the Telegram chat ID (text)
- `activeReqId` — optional bool column; only members with `val == '1'` or `'true'` are included
- `roleReqId` + `roleIds` — optional ref-column filter; only members whose role is one of the given record IDs are included
- If `recipients.fromTable` is set, `chatId` is ignored.

**Multi-button inline keyboard (`buttons[]`):**

Render several action buttons (one per row). Falls back to the legacy `buttonText`/`buttonReqId`/`buttonValue` triple if `buttons` is absent.

```json
{
  "type": "send_telegram",
  "message": "Заказ №{{val}} готов к сборке",
  "buttons": [
    { "text": "Взять в работу", "reqId": 378, "value": "393",
      "fromValue": "392", "requireRole": [420, 419], "trackerReqId": 264129 },
    { "text": "Отменить", "reqId": 378, "value": "399",
      "requireRole": 419,
      "requireStatusBelow": { "reqId": 378, "value": "395" } }
  ]
}
```

Per-button modifiers (all optional):

- `fromValue` — CAS guard. Only set the field if its current value equals this. Lost race → callback responds «Статус уже изменился».
- `requireRole` — record-id (or array of record-ids) of the role(s) the sender must have. Enforced **twice**: at send time the button is hidden from recipients whose `Сотрудник.Роль` does not match (so only the right people see it in their keyboard); at click time the callback handler re-checks. Pass a single number (`420`) for one-role gating or an array (`[420, 419]`) when the action should be available to multiple roles — e.g. when a Пчеловод may also collect an order if no Сборщик is free. Encoded as `r=420` or `r=420,419` on the wire. Set up in `portal_config.notifications.telegramEmployeeTable`.
- `requireStatusBelow` — `{ reqId, value }`. Numeric comparison; the click is rejected if `obj[reqId] >= value`. Used e.g. for «отмена возможна только до В доставке».
- `requireParentStatusBelow` — `{ reqId, value }`. Same as `requireStatusBelow`, but checks the **parent record** (`obj.parentId`). Use for child-record buttons whose availability depends on parent status, e.g. «отменить дозаказ нельзя, если родительский заказ уже в доставке». Emits `lp=<reqId>:<value>` modifier.
- `trackerReqId` — record the sender's employee record-id into this field (only if the field is currently empty). Used for «кто взял в работу».

**Per-recipient keyboard filtering.** When `recipients.fromTable` is used, the keyboard is rebuilt for each recipient before sending:

- Button with no `requireRole` → visible to everyone.
- Button with `requireRole = N` → visible only to recipients whose `Сотрудник.Роль = N`.
- Button with `requireRole = [A, B, …]` → visible to recipients whose role is any of the listed ids.
- If no buttons survive filtering for a recipient, the message is sent **without** `reply_markup` (plain text).
- `childButtons` block supports its own `requireRole` to hide the whole checkbox block (toggles + complete) from non-matching recipients. `completeButton.requireRole` filters that single button independently within an otherwise-visible block.

For the legacy single-`chatId` path (no role lookup), buttons with `requireRole` are **hidden** — role-gated flow is opt-in via the broadcast pattern. Buttons without `requireRole` always show.

**Child checkbox buttons (`childButtons`):**

Render one toggle button per child record, optionally followed by a "complete" button that fires only when every child's toggle is `'1'`.

```json
{
  "type": "send_telegram",
  "message": "Заказ №{{val}} — отметьте собранные позиции",
  "childButtons": {
    "childTypeId": 315,
    "toggleReqId": 300,
    "qtyReqId": 366,
    "completeButton": {
      "text": "📦 Собран", "reqId": 378, "value": "394",
      "fromValue": "393", "requireRole": 420, "trackerReqId": 264129
    }
  }
}
```

- `childTypeId` — type of child records (e.g. «Товары заказа»). Children are looked up by `up = <trigger object id>, t = childTypeId`.
- `toggleReqId` — the bool field per child that the button flips.
- `qtyReqId` — optional. If set, the button label is `«✅ <name> ×<qty>»`.
- `requireRole` — optional. Hide the whole block (toggles + complete) from recipients whose role does not match. Accepts a single id or an array (`[420, 419]`) when multiple roles may check items / fire the complete button.
- `completeButton` — optional final button; same modifier shape as a normal button (including its own `requireRole` for additional gating). Callback verifies every child's `toggleReqId` is `'1'`/`'true'` before applying.

**Callback handling.**

Callback data format: `<op>:<args>[;<mod>=<val>...]`. Ops:

| op | args | meaning |
|----|------|---------|
| `u` | `objId:reqId:value` | set EAV field |
| `t` | `objId:reqId` | flip bool (`'1'`↔`'0'`); on success the message keyboard is re-rendered with the icon swapped (⬜↔✅) for that button only |
| `c` | `parentId:reqId:value:childTypeId:toggleReqId` | set parent reqId=value only if every child has toggle ≡ true |
| `n` | `automationId:screenId[:page]` | navigate to a screen in a multi-screen keyboard; `screenId = "_back"` pops the Redis navigation stack and returns to previous screen. Message text and keyboard are updated in-place via `editMessageText` |

Modifiers: `r=<roleId>`, `f=<expected>`, `l=<reqId>:<maxVal>`, `lp=<reqId>:<maxVal>` (same check applied to the parent record via `up`), `T=<reqId>`.

Sender resolution requires `portal_config.notifications.telegramEmployeeTable = { typeId, chatIdReqId, roleReqId? }`. Without it, `requireRole` cannot match and click is denied with «Вы не зарегистрированы как сотрудник».

Values must not contain `:` or `;` — use numeric IDs only.

**Broadcast keyboard sync.** When `recipients.fromTable` sends the same message to N recipients, each gets a separate `(chat_id, message_id)`. After one click, the dispatcher fans the keyboard transition out to the other N-1 messages too (cleared on `u`/`c`, icon flipped on `t`). Implementation lives in `portal/telegram-broadcast.js` and a per-workspace satellite `_v2_tg_sent_messages`. See `backend/docs/modules/portal.md` → «Broadcast keyboard sync» for details and trade-offs.

**Multi-screen navigation (`screens`):**

Define a hierarchy of named screens, each with its own text and buttons. The "main" screen is shown initially. Buttons with `go` navigate between screens; `go: "_back"` returns to the previous screen. Navigation state is stored in Redis (`tg_nav:{chatId}:{botId}`, TTL 1 hour).

```json
{
  "type": "send_telegram",
  "screens": {
    "main": {
      "text": "Главное меню",
      "buttons": [
        { "text": "⚙️ Настройки", "go": "settings" },
        { "text": "📋 Заказы", "go": "orders" }
      ]
    },
    "settings": {
      "text": "Настройки",
      "buttons": [
        { "text": "🔔 Уведомления", "go": "notifications" },
        { "text": "⬅️ Назад", "go": "_back" }
      ]
    }
  }
}
```

- `go: "screenId"` — navigates to another screen in the same `screens` config
- `go: "_back"` — pops the navigation stack, returns to the previous screen (restores page + filter)
- `goBackTo: "screenId"` — named back: jumps to a specific screen, clearing stack entries above it
- `goFilter: "value"` — adds a filter value to navigation (e.g., status filter)
- Each screen can have `url` buttons (open links) and regular `reqId`/`value` buttons (EAV mutations)
- When `screens` is present, `message` is optional — text is taken from the active screen's `text`
- Template variables: `{{_screen}}` (also available as `{{screen}}`), `{{_page}}`, `{{_items}}`, `{{_breadcrumb}}` (nav trail: «Screen1 > Screen2 > Current»)
- Navigation stack stores `{automationId, screenId, page, filterValue}` — full context preserved on back
- Screens can be paginated with `listSource` for dynamic record lists

**listSource — dynamic paginated lists:**

```json
"orders": {
  "text": "Заказы (стр. {{_page}})",
  "listSource": {
    "queryMode": "table",
    "typeId": 311,
    "pageSize": 6,
    "filterReqId": 378
  },
  "listButton": {
    "text": "Заказ №{{name}}",
    "go": "detail"
  },
  "buttons": [
    { "text": "⬅️ Назад", "go": "_back" }
  ]
}
```

- `queryMode`: `"table"` — root records (t=typeId, up=1) | `"children"` — child records (up=parentId, t=typeId)
- `typeId` — table type to list
- `pageSize` — items per page (default 8)
- `filterReqId` — EAV field for filtering (used with `goFilter` on buttons)
- `listButton.text` — template for item buttons (`{{name}}`, `{{id}}`)
- `listButton.go` — target screen when item is clicked; `{{id}}` and `{{val}}` plus EAV requisites (`{{req_NNN}}`) are available in the detail screen

**replyKeyboard — persistent bottom navigation bar (ReplyKeyboardMarkup):**

```json
"main": {
  "text": "📋 Меню",
  "replyKeyboard": [
    [{"text": "📋 Заказы", "go": "orders"}, {"text": "📦 Сборка", "go": "orders", "goFilter": "393"}],
    [{"text": "🍯 Товары", "go": "products"}]
  ]
}
```

- Replaces user's default keyboard with a persistent bottom bar — always visible, unlike inline buttons
- Button press sends plain text (NOT callback) → handled by `tryHandleReplyKeyboardNav` — scans all active automations for matching `replyKeyboard` button
- Each button must have `text` (display + match text) and `go` (target screen ID)
- Optional: `goFilter` to pre-filter the target screen
- ReplyKeyboard stays visible during inline-keyboard navigation — hybrid: bottom bar for section switch, inline for contextual actions
- Screen with `replyKeyboard` cannot also have inline `buttons`/`listSource` on the SAME message (Telegram API: one reply_markup type per message). Sub-screens use inline keyboards.
- `resize_keyboard: true` by default (compact mode). Set `replyResize: false` to disable.
- `is_persistent: true` by default (keyboard stays visible when scrolling). Set `replyPersistent: false` to disable.
- `{{_filter_label}}` resolves dynamically from EAV: if `goFilter` is a numeric record ID, its display name is loaded from the database. No hardcoded label maps — works for any workspace and any lookup table.

### delete_telegram_message

Deletes a previously sent Telegram message. Config: `{ botToken, chatId, messageId }`

```json
{
  "type": "delete_telegram_message",
  "botToken": "7812345678:AAH...",
  "chatId": "-1001234567890",
  "messageId": "{{_tg_message_id}}"
}
```

### send_chat_action

Sends chat action (typing indicator). Config: `{ botToken, chatId, action }`

```json
{
  "type": "send_chat_action",
  "botToken": "7812345678:AAH...",
  "chatId": "-1001234567890",
  "action": "typing"
}
```

### send_notification_to_group

Sends a topbar notification to every member of a table whose login field is filled.

```json
{
  "type": "send_notification_to_group",
  "typeId": 313,
  "usernameReqId": 151139,
  "title": "Заказ №{{val}} готов к сборке",
  "body": "Адрес: {{req_339}}",
  "filter": { "reqId": 315, "val": "Сборщик" }
}
```

- `typeId` — table to read members from
- `usernameReqId` — column reqId that stores the Integram username
- `filter` (optional) — only notify members where `reqId == val`
- `title` / `body` — support `{{template}}` variables from the trigger context

### run_script

Execute JavaScript in an isolated V8 sandbox (`isolated-vm`).

**Action config:**
```json
{
  "type": "run_script",
  "script": "output(row['Price'] * row['Qty'])",
  "outputReqId": 123,
  "timeoutMs": 30000
}
```

**Script API:**
- `row` — record fields as `{ ID, parentId, FieldName: value, ... }` (loaded from EAV if objId is set; empty `{}` otherwise). `parentId` is the parent object ID (null for root-level records)
- `output(value)` — set return value (string)
- `setField(reqId, value)` — write to a field on the current record. **Requires objId** — silently skipped if no object context (e.g. manual trigger without object).
- `fetch(url, opts)` — HTTP request, returns `{ status, body, ok }`. SSRF-protected: localhost, private IPs, `.local`, `.internal` are blocked.
- `ai(prompt, model?)` — LLM call (model: `"fast"` default), returns response text
- `query(typeId, opts)` — list records, returns `[{ id, value, parentId, typeId, order }]` (metadata only, no field values; use `getRecord(id)` for full record)
- `getRecord(id)` — get single record with all fields
- `queryPage(typeId, opts)` — the same read as `query()`, but keeps the pagination metadata: `{ rows, total, page, pageSize, hasMore, nextCursor }`. Use it whenever the table may hold more rows than `pageSize` — `query()` returns a bare array and its truncation is silent. Each page bills the `query` counter.
- `createRecord(typeId, { name, fields, parentId })` — create record, returns new ID. `fields` = `{ reqId: value }`; `parentId` makes it a child record of that object (omit for root level)
- `updateRecord(id, { fields })` — update record fields
- `deleteRecord(id)` — delete record
- `moveChildren(...)` — re-parent child records
- `createDocument(...)` / `listDocuments(...)` / `getDocument(...)` — documents through the documents service, with the caller's own RBAC. The two reads bill the `query` counter, the create bills `mutation`.
- `kag_search(...)` / `kag_relations(...)` — read the knowledge graph (bill `query`)
- `kag_import_entities(...)` / `kag_import_relations(...)` — **write** to the knowledge graph. Bill `mutation`. **Require the `admin` role** — the platform keeps graph import behind `requireRole('admin')` on the REST side (`POST /:db/swarm-memory/kag/import-*`), and the bridge does not widen that.
- `search_decisions(...)` — search decisions (bills `query`)
- `delegateToAgent(...)` — hand a task to an agent
- `sendTeamchatMessage(...)` — post into teamchat
- `browse(query, source?)` — search marketplace prices via browser service, returns `[{ name, price, url, source }]`. Default source: `'komus'`. Counts against the fetch rate limit.
- `console.log(...)` — captured in `logs` array

The authoritative list of bridged names is the `__BRIDGED` array in
`automations/isolated-runner.js` — a name on that list that was **not** injected
into this particular sandbox is replaced by a stub that throws
`<name>: мост не проброшен в эту песочницу`, so an unavailable bridge fails
loudly instead of reading as `undefined`.

**Limits:** 128 MB RAM; 30 query / 50 mutation / 50 fetch / 10 AI / 20 agent /
50 teamchat calls per execution (`SANDBOX_PROFILES.automation` in
`utils/sandbox-profiles.js`).

**The timeout is the caller's, and 60 s is only one caller's default.**
`executeIsolated` takes `timeoutMs` and falls back to the profile's 60 s only
when the caller passes nothing; **it does not cap the value from above**. The
three callers differ:

| Caller | Timeout it passes |
|--------|-------------------|
| `execution.js` (`run_script` action) | `action.timeoutMs`, defaulting to 60 s |
| `batch.js` (batch run) | `scriptAction.timeoutMs`, defaulting to **300 s** |
| `ai/agent/index.js` (the `run_script` tool) | `Math.min(payload.timeoutMs \|\| 60000, 60000)` — hard 60 s ceiling, the only path that clamps |

**Environment variables set after execution:**
- `_script_result` — output value (not set on error)
- `_script_logs` — joined log lines (only set when logs exist)
- `_script_error` — error message (only set on failure; `_script_result` and `_script_logs` are NOT set when error occurs)

**AI/MCP tool:** Also available as `run_script(script, typeId, objectId?, timeoutMs?)` tool (TIER_HIGH, requires confirmation).

### run_server_function

Call a codespace server function — a file `api/<fn>.js` in a workspace git repository. This is how repo code is put on a schedule: before this action, `executeServerFunction()` had a single caller, `POST /:db/portal/api/fn/:repo/:name`, so functions ran only in response to an HTTP request from the portal.

**Action config:**
```json
{
  "type": "run_server_function",
  "repo": "monitoring",
  "fn": "check-sources",
  "args": { "org": "{{req_501}}" },
  "resultVar": "mon",
  "idempotencyKey": "daily-{{today}}",
  "idempotencyMinutes": 1
}
```

- `repo`, `fn`, `idempotencyKey` and every string value in `args` go through normal `{{...}}` template resolution.
- The function runs under the automation's user with `role: 'admin'` and `source: 'automation'`.

**Sandbox:** identical to the portal path — `codespace/server-fn-executor.js`. 64 MB, 15 s (5 s when the function declares no capabilities), capabilities declared by a `// capabilities: query, fetch` comment inside the function file. This action declares no limits of its own; see ADR-023.

**Idempotency (on by default):** the unit is `repo + fn + args + record`, stored in `_v2_automation_cooldowns`. Within `idempotencyMinutes` (default `1`) the same invocation runs once. Set `0` to disable.

> Keep `idempotencyMinutes` **below the schedule interval** — a larger value silently skips runs.

The claim is taken by a single `INSERT ... ON CONFLICT DO UPDATE ... WHERE last_fired_at < NOW() - interval RETURNING`, so concurrent runs cannot both win. **A failed call releases its claim**, so the next scheduled run retries — the guard cannot latch shut after an outage. There is no session-scoped lock, so a claim left by a crashed process expires with the window instead of wedging every later attempt (the failure mode fixed in `create_cdek_shipment`).

**Environment variables set after execution:**
- `{{_server_fn_skipped}}` — `'1'` when the call was suppressed as a duplicate
- `{{_server_fn_error}}` — error message on failure
- `resultVar` — JSON-encoded return value on success

### update_related_records

Declarative cross-table update. Traverses child records, matches refs, updates a numeric field in a target table. Declarative alternative to `run_script` for patterns like inventory deduction.

```json
{
  "type": "update_related_records",
  "childTypeId": 315,
  "matchRefReqId": 381,
  "targetTypeId": 312,
  "targetMatchReqId": 379,
  "targetFieldReqId": 351,
  "sourceFieldReqId": 361,
  "operation": "subtract"
}
```

- `childTypeId` — child table type (e.g. Order Items)
- `matchRefReqId` — ref column in child pointing to the shared entity (e.g. Product)
- `targetTypeId` — target table (e.g. Inventory)
- `targetMatchReqId` — ref column in target pointing to the same entity (e.g. Product)
- `targetFieldReqId` — numeric field to update (e.g. Stock)
- `sourceFieldReqId` — numeric field in child used as the value (e.g. Quantity)
- `operation` — `subtract` | `add` | `set`

Writes go through `bus.emit('object.updated')` — triggers webhooks, graph, audit.

### `reconcile_cdek`

Periodic CDEK status reconciliation. Polls CDEK API for orders with UUID in non-terminal statuses, updates EAV when CDEK reports delivery/pickup/return.

**Config:**
| Field | Type | Description |
|-------|------|-------------|
| `connectorId` | integer | ID of the CDEK connector record (type=cdek) |
| `resultVar` | string? | Optional env variable to store result stats |

**Trigger:** `schedule` with cron (recommended: once daily `0 6 * * *`)

**Result:** `{ checked, updated, skipped, errors }`

**Status mapping:** Uses `RECONCILE_STATUS_MAP` from `cdek/actions.js` (subset of `CDEK_STATUS_MAP` — only terminal/significant statuses):
- `DELIVERED` → `Доставлен`
- `READY_FOR_PICKUP` → `В пункте выдачи`
- `RETURNED_TO_SENDER_CITY` → `Отменён`
- All other CDEK statuses → skipped (already mapped as В доставке by webhook)

**Idempotency:** Skips orders whose status already matches CDEK. Safe to run multiple times.

**Example automation:**
```json
{
  "trigger": { "type": "schedule", "cron": "0 6 * * *" },
  "actions": [{ "type": "reconcile_cdek", "connectorId": 1003351 }]
}
```

## Template Variables

Action fields (value, message, requisites values, parentId) support `{{variable}}` placeholders resolved at runtime via `resolveVal()`.

| Variable | Description |
|----------|-------------|
| `{{id}}` | Trigger object ID |
| `{{val}}` | Trigger object display name |
| `{{typeId}}` | Trigger object type ID |
| `{{parentId}}` | Trigger object parent ID |
| `{{created_by}}` | Username who created/updated the object |
| `{{req_NNN}}` | Value of field with reqId=NNN on the trigger object |
| `{{NOW}}` | Current ISO datetime (e.g. `2026-05-09T19:54:02.752Z`) |
| `{{TODAY}}` | Current date (e.g. `2026-05-09`) |
| `{{_trigger_user}}` | Username of the user who triggered the event |
| `{{_trigger_email}}` | Email of the user who triggered the event |
| `{{body_*}}` | Webhook body fields (for `on_webhook` triggers, e.g. `{{body_status}}`) |
| `{{metric}}`, `{{value}}`, `{{source_id}}`, `{{ts}}` | Metric context (for `on_metric_threshold`/`on_metric_silence` triggers) |
| `{{silence_minutes}}` | Minutes since the metric was last seen (for `on_metric_silence` triggers) |
| `{{doc_id}}`, `{{doc_title}}`, `{{doc_action}}`, `{{doc_status}}` | Document context (for `on_document` triggers) |

Built in `buildEnv()` (`service.js`). Available in all action types.

## Condition

Optional filter on the trigger event. Two modes:
- `formula`: expression using `[FieldName]` column references
- `ai`: LLM evaluates a prompt with object context

**The `formula` mode is not the bare formula engine — it goes through
`evalCondition` (`automations/helpers.js`), which is deliberately stricter and
**throws** rather than answering "no".** Four cases raise:

| Case | Message |
|------|---------|
| Empty expression | `выражение не задано` |
| `[Имя]` with no such variable in the environment | `в окружении нет переменной «…»` |
| Expression does not parse | `выражение «…» не разбирается — …` |
| A comparison against a non-numeric value (`>`, `<`, …) | `сравнивает нечисловое значение` — for a string use `==` or a `switch` action |
| Parsed but yielded nothing (division by zero, unknown function) | `разобрано, но не дало значения` |

**A throw is a failed run, not a skipped one.** `execution.js` catches it,
writes the run as `status: 'error'` with the message, and returns
`{ ok: false, error }`. Only a cleanly computed falsy value (`0`, `'0'`,
`false`, `''`) is logged as `skipped`. The distinction is the point: "did not
match" and "is broken" must not read alike in the run log — a mistyped field
name used to look like an automation that had worked and decided not to fire.

The same function backs the `if_else` action, under the label `if_else`.

## Worker (`worker.js`)

BullMQ consumer on `automations` queue (concurrency 10). Executes actions in sequence. On failure, retries with exponential backoff. Results written to `_v2_automation_runs`. Error message from failed actions is captured and passed to `logRun()` as `status: 'error'`.

**Context carried through the queue.** The job payload and the worker's
`runCtx` are both built by hand, so a field must be added in **both** places or
it silently disappears between enqueue and execution. Telegram context goes
through `pickTelegramCtx()` / `buildRunCtx()` (exported from `worker.js`) for
exactly that reason — `runCtx` is what reaches `buildEnv`, and therefore the
`{{...}}` template vars.

The two callers want opposite defaults for absent keys. The payload passes
`fillMissing: true` — `undefined` vanishes in the Redis JSON round-trip, `null`
survives. `runCtx` omits them: `buildEnv` gates the command block on
`_tgCommand !== undefined`, so a null there would hand every non-Telegram
automation a set of empty `_command` / `_args` vars.

**The acting user goes through with `role` and `grants`.** They are part of the
serialized `ctx.user`: without them every permission check on the worker side
resolves to `BARRED`, so an action that writes a file (`telegram_download_file`)
or touches a guarded table fails with 403 — but only when Redis is up, since the
in-process path passes the user object as is. A job with no user at all falls
back to `queue-worker` with admin rights, which made the truncated-user case
strictly worse than the missing-user one.

⚠ This makes both paths behave the same, and that exposes an older decision:
`dispatchers.js` builds the acting user for a Telegram trigger with a hardcoded
`role: 'admin'` (`:175`, `:260`), and `repoGrant` grants `WRITE` to that role.
So anyone who writes to the bot drives file writes with administrator rights —
through the queue as well now, not only in-process. The queue used to block this
by accident, not by design. Narrowing it means giving the Telegram acting user a
role of its own, and that is a separate change.

Cron automations: registered with BullMQ as recurring jobs. Sync'd on `automation.saved/deleted` events. `syncCronJobs()` runs on startup — iterates ALL workspaces, reads `_v2_automations` with `active=TRUE` and `trigger.type='schedule'`, registers repeatable jobs. Orphaned jobs removed.

### Scheduler user

Cron-triggered automations run as `{ username: 'scheduler', grants: {}, roleId: 0, role: 'admin' }`. The `role: 'admin'` is required — without it, `checkGrant()` denies access to connectors (grants is empty, no role bypass). The `run_connector` action builds `mockUser` with `role: ctx.user?.role || 'admin'` fallback for the same reason.

### Stalled jobs

When the integram process restarts, active BullMQ jobs from the previous worker become "stalled" — they occupy worker slots but never complete. If many workspaces have schedule automations that fail (e.g. grants bug), stalled jobs accumulate and block all 10 worker slots.

**Automatic cleanup:** `initAutomationsWorker()` cleans stalled jobs on startup — any active job older than 2 minutes is moved to failed and purged before the new worker registers. This prevents stalled jobs from blocking production cron after pm2 restart.

## Listeners (`listeners.js`)

Subscribes to event bus:
- `object.created` → enqueues `on_create` automations. Emitted by: REST object create, AI agent `createObject`, connector import (new record)
- `object.updated` → enqueues `on_update` automations. Emitted by: REST object update, AI agent `updateObject`, connector import (existing record with changed field values). Script button writes go through REST `PATCH /objects/:id` from the browser, so they emit `object.updated` normally.
- `object.deleted` → enqueues `on_delete` automations
- `object.created` with `source: 'form'` → enqueues `on_form_submit` automations (matched via `on_create` + `ctx.source === 'form'`)
- `approval.resolved` → resumes suspended `request_approval` actions
- `automation.saved/deleted` → syncs cron schedule with BullMQ

## Idempotency

`X-Idempotency-Key` header supported on POST. Cached results served for 30 seconds.

## DB Tables

- `_v2_automations` (per-workspace) — automation definition, trigger, condition, actions (JSONB). Column `is_system BOOLEAN` marks automations seeded by the system at workspace creation (informational only — user can edit/delete freely).
- `_v2_automation_runs` (per-workspace) — execution log: status, output, error, duration
- `_v2_automation_cooldowns` (per-workspace) — PK `(automation_id, cooldown_key)`, `last_fired_at`. Backs both the metric-trigger cooldowns (`on_metric_threshold` / `on_metric_silence`) and the `run_server_function` replay guard.

## System Automations

Two automations are seeded automatically at workspace creation via `seedSystemAutomations()`:

| Name | Trigger | Purpose |
|------|---------|---------|
| AI-анализ данных | `ai_analysis` cron `0 3 * * *` | Nightly LLM analysis of all tables, generates insights |
| Здоровье автоматизаций | `schedule` cron `0 4 * * *` | Checks for active automations that haven't fired in 14+ days |

`is_system = true` automations show a "Системная" badge in the UI. Users can delete them and restore via `POST /seed-system` (or the button in WorkspaceEdit → AI tab). `seedSystemAutomations()` is idempotent — skips automations whose name already exists with `is_system=true`.

**System automations are created with `active = FALSE` by default** — they are opt-in. The workspace owner must explicitly enable them. This prevents LLM workload from running in every workspace without consent.

Existing workspaces that were created before this change will simply not have system automations at all — nothing to worry about.

## ai_analysis → agent_memory (vector recall)

After generating insights, `runAiAnalysis()` writes each one to `agent_memory` via `swarmWrite`:

```
agent_id     = '$shared'          ← found by all agents via hybridSearch
scope        = 'shared'
key          = 'insight:<dismissed_type_key>:<timestamp>'
value        = '<headline>. <reasoning>'
tags         = ['scan', 'ai_analysis', 'source_type:<typeId>']
temporal_type = 'permanent'
```

`swarmWrite` handles semantic deduplication automatically (cosine threshold 0.85 + LLM ADD/NOOP/UPDATE decision). Insights are recalled by the agent via `recallFacts()` without any extra code — they appear in the agent's memory context when the question is semantically related.
