# Module: telegram-bots

**Path:** `src/api/v2/modules/portal/` (bot management lives in portal service/router; dedicated module extraction planned)
**Base URL:** `/api/v2/:db/portal/api/bots`
**Auth:** Admin JWT required (workspace admin role).
**Frontend UI:** `/:db/telegram-bots` (list), `/:db/telegram-bots/:id` (detail with 5 tabs)

## Purpose

Multi-bot Telegram constructor. Each workspace can have multiple bots. Bots are a facade over the automation system — bot reactions (commands, keywords) and notifications are stored as automations linked via `botId`.

## Table

`_v2_tg_bots` (global, not per-workspace schema):

| Column | Type | Description |
|--------|------|-------------|
| id | BIGINT | Auto-increment PK |
| db | VARCHAR(64) | Workspace slug (e.g. `usadba_3`) |
| name | VARCHAR(255) | Display name |
| username | VARCHAR(128) | Telegram @username (without @) |
| token | VARCHAR(256) | BotFather token |
| enabled | BOOL | Webhook handler ignores disabled bots |
| webhook_secret | VARCHAR(64) UNIQUE | `sha256(token).slice(0, 32)` — computed on create/update |
| config | JSONB | Bot config (see below) |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

## REST Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/bots` | List bots (excludes token) |
| POST | `/api/bots` | Create bot + auto-register Telegram webhook |
| GET | `/api/bots/:id` | Bot detail (excludes token) |
| PATCH | `/api/bots/:id` | Update bot; token change re-registers webhook |
| DELETE | `/api/bots/:id` | Delete bot + deregister webhook |
| POST | `/api/bots/:id/sync` | Sync config to Telegram API (commands, description, menu button) |
| GET | `/api/bots/:id/status` | Get bot info (getMe) + webhook status (getWebhookInfo) |
| POST | `/api/bots/:id/test` | Send test message. Body: `{ chatId, text }` |

## Bot Config Schema

Stored in `config` JSONB column:

| Field | Type | Description |
|-------|------|-------------|
| `commands` | Array | `[{ command: "/status", description: "...", action: "reply"\|"automation", replyText?: "...", automationTrigger?: "..." }]`. Reserved hardcoded commands: `/neworder` (enters new order input flow) and `/cancel` (cancels any pending input state). These are handled before automation dispatch. |
| `description` | String | Bot description (max 512 chars, synced to Telegram, shown on /start screen) |
| `shortDescription` | String | Short description (max 120 chars, synced, shown in search) |
| `welcomeMessage` | String | Auto-reply on bare `/start` without deep-link (HTML) |
| `menuButton` | Object | `{ type: "commands"\|"web_app"\|"default", text?: "...", web_app?: { url: "..." } }` (synced) |
| `keywords` | Array | `[{ trigger: "hello,hi", replyText: "Hello!", matchMode?: "contains"\|"exact" }]` — legacy, prefer automations |
| `employeeTable` | Object | `{ typeId, chatIdReqId, roleReqId }` — for inline-button role resolution in callback handler |
| `startScreen` | String | Automation action override for the starting screen (e.g. `"main"`) |
| `botName` | String | Portal config display name, synced to Telegram via `setMyName` |
| `nameReqId` | Number | listSource config: resolves button label from a requisites field |

## Bot <-> Automations

Bot logic lives in automations, linked by `botId`:

**Reactions (incoming):**
- Commands: automation with `trigger: { type: "on_telegram_command", command: "/status", botId: N }`
- Keywords: automation with `trigger: { type: "on_telegram_message", pattern: "hello,hi", matchMode: "contains", botId: N }`
- Intake (every message, text and media): same trigger **without** `pattern` — optionally narrowed by `messageTypes: ["voice","document"]`. Runs alongside keyword rules. See `automations.md`.

**Notifications (outgoing):**
- `send_telegram` action with `botId: N` — executor resolves token from `_v2_tg_bots` via `getBotInternal()`

**Template vars** available in automation `buildEnv`: `_command`, `_args`, `_chat_id`, `_from_username`, `_from_first_name`, `_tgMessage`, `_items` (list items for child tables, joined with newlines). `on_telegram_message` adds the full message context — author id, message id, date, reply-to, attachment — listed in `automations.md`.

**Filtering:** `GET /automations?botId=N` returns only automations for a specific bot.

## Token Resolution Order

`send_telegram` action in `automations/service.js`:

1. `action.botToken` — hardcoded (not recommended)
2. `action.botId` → `getBotInternal(pool, db, botId)` → `_v2_tg_bots`
3. `action.integrationMode: "table"` → EAV lookup (legacy fallback)

## Webhook

Shared endpoint: `POST /api/v2/portal/telegram/:secret` (global, no `:db` prefix).

**Resolution order** (`resolveBotBySecret` in `portal/service.js`):
1. `_v2_tg_bots WHERE webhook_secret = ?` — multi-bot path
2. `_v2_portal_config WHERE telegram_secret = ?` — legacy fallback

**Message routing:**
- `callback_query` → `handleTelegramCallback()` (inline keyboard button presses)
- Bare `/start` → sends `welcomeMessage` if configured
- `/start <token>` → deep-link auth flow (portal login)
- Custom `/commands` → `dispatchTelegramCommand()` to matching automation
- Non-command text → `dispatchTelegramMessage()` to keyword and intake automations, legacy fallback to `botConfig.keywords[]`
- Media messages (voice, document, photo, …) → `dispatchTelegramMessage()`; portal conversational handlers are text-only and are skipped
- `edited_message` → `dispatchTelegramMessage()` with `_edited = 1`
- `my_chat_member` → `dispatchBotChatMember()` — bot added/removed from group, blocked/unblocked by user

`my_chat_member` and `edited_message` arrive as updates **without** a `message`
field, so their handlers run before the `if (!message) return` guard.
- Contact share → phone-based portal auth

Strips `@botname` suffix from commands in groups.

## Sync

`POST /api/bots/:id/sync` calls Telegram API:
- `setMyCommands` / `deleteMyCommands`
- `setMyDescription` / `setMyShortDescription`
- `setChatMenuButton`

## Migration

**Auto-migration:** `ensureDefaultBot(pool, db)` — lazy on first `listBots()`. If workspace has no bots in `_v2_tg_bots` but has legacy `telegramBotToken` in portal config, creates a "Default Bot" record.

**Manual migration** (preserving legacy fallback):
```sql
-- 1. Create bot record
INSERT INTO _v2_tg_bots (db, name, username, token, enabled, webhook_secret, config)
VALUES ('<db>', '<name>', '<username>', '<token>', true,
  '<sha256(token).slice(0,32)>',
  '{"commands": [], "employeeTable": {"typeId": ..., "roleReqId": ..., "chatIdReqId": ...}}');

-- 2. Add botId to automations (keep legacy fields as fallback)
UPDATE <db>._v2_automations
SET actions_cfg = (
  SELECT jsonb_agg(
    CASE WHEN elem->>'type' = 'send_telegram'
    THEN elem || '{"botId": <id>}'::jsonb
    ELSE elem END
  ) FROM jsonb_array_elements(actions_cfg) AS elem
) WHERE id IN (...);
```

## AI Tools (27 MCP tools)

### Bot Management

| Tool | Tier | Description |
|------|------|-------------|
| `list_telegram_bots` | LOW | List all bots for workspace |
| `create_telegram_bot` | HIGH | Create bot + register webhook |
| `update_telegram_bot` | HIGH | Update bot; token change re-registers webhook |
| `delete_telegram_bot` | HIGH | Delete bot + deregister webhook |
| `sync_telegram_bot` | HIGH | Sync config to Telegram API |
| `get_telegram_bot_status` | LOW | getMe + getWebhookInfo |
| `test_telegram_bot` | MEDIUM | Send test message to chatId |

### Messaging

| Tool | Tier | Description |
|------|------|-------------|
| `telegram_forward_message` | MEDIUM | Forward or copy a message between chats |

### Payments

| Tool | Tier | Description |
|------|------|-------------|
| `telegram_send_invoice` | HIGH | Send payment invoice (Stars or provider) |
| `telegram_create_invoice_link` | MEDIUM | Create payment URL (no chat needed) |

### Chat Administration

| Tool | Tier | Description |
|------|------|-------------|
| `telegram_ban_member` | HIGH | Ban user in group/channel |
| `telegram_unban_member` | HIGH | Unban user |
| `telegram_restrict_member` | HIGH | Restrict user permissions (mute, read-only) |
| `telegram_promote_member` | HIGH | Promote/demote to admin |
| `telegram_approve_join` | HIGH | Approve pending join request |
| `telegram_decline_join` | HIGH | Decline pending join request |
| `telegram_pin_message` | MEDIUM | Pin a message |
| `telegram_unpin_message` | MEDIUM | Unpin one or all messages |
| `telegram_get_chat` | LOW | Get chat info (title, type, members) |
| `telegram_get_chat_member_count` | LOW | Get member count |
| `telegram_create_invite_link` | MEDIUM | Create invite link with options |

### Stories

| Tool | Tier | Description |
|------|------|-------------|
| `telegram_post_story` | HIGH | Publish story to channel |
| `telegram_edit_story` | HIGH | Edit posted story |
| `telegram_delete_story` | HIGH | Delete story |

### Business API

| Tool | Tier | Description |
|------|------|-------------|
| `telegram_get_business_connection` | LOW | Get business connection info |
| `telegram_set_business_bio` | HIGH | Set business account bio |
| `telegram_set_business_name` | HIGH | Set business account name |

## Webhook Hardening

See [portal.md](portal.md#telegram-webhook-hardening) — three layers protect the shared `/portal/telegram/:secret` endpoint:
1. Header verification (`X-Telegram-Bot-Api-Secret-Token`)
2. Update_id deduplication (Redis, 24h TTL)
3. Outbound rate limiting (30/sec global, 1/sec per chat)

## Multi-Screen Navigation (screens)

Inline keyboards can be organized into named screens with hierarchical navigation. Defined in the `send_telegram` automation action via `action.screens`.

**Screen config format:**
```json
{
  "type": "send_telegram",
  "screens": {
    "main": {
      "text": "Welcome!",
      "buttons": [
        { "text": "Settings", "go": "settings" },
        { "text": "Orders", "go": "orders" }
      ]
    },
    "settings": {
      "text": "Settings",
      "buttons": [
        { "text": "Back", "go": "_back" }
      ]
    }
  }
}
```

**How it works:**
- `main` screen is shown when the automation triggers
- Buttons with `go: "screenId"` generate `n:automationId:screenId:page` callback_data (always includes page number, default `:1`)
- On click, the callback handler (`telegram-callback.js`) loads the target screen config, renders the new keyboard, and updates the message in-place via `editMessageText`
- Navigation stack is stored in Redis: `tg_nav:{chatId}:{botId}` → JSON array `[{automationId, screenId}]`, TTL 1 hour
- `go: "_back"` pops the stack and returns to the previous screen
- If stack is empty when navigating back, the keyboard is cleared

**Callback op codes (`telegram-callback.js`):**

| op | args | meaning |
|----|------|---------|
| `u` | `objId:reqId:value` | set EAV field |
| `t` | `objId:reqId` | flip bool (`'1'`↔`'0'`); re-renders keyboard with icon swapped (⬜↔✅) |
| `c` | `parentId:reqId:value:childTypeId:toggleReqId` | set parent reqId=value only if every child toggle ≡ true |
| `n` | `automationId:screenId[:page]` | navigate to a named screen; `_back` pops nav stack |
| `b` | `automationId:screenId` | named back (goBackTo) — jump to a specific screen, clearing stack above it |
| `at` | `objId:childTypeId:toggleReqId` | assembly toggle — flip bool on a child record |
| `mu` | `objId` | multi-field update — writes multiple EAV fields from payload in one operation |
| `di` | `objId:reqId` | dimensions input — prompts user for dimension value, writes parsed result |
| `pl` | `objId` | print CDEK label — triggers label printing flow for a shipment |
| `sa` | `objId:reqId` | stock adjust — adjust a numeric stock field by a delta |
| `si` | `objId:reqId` | stock input — set stock field to an absolute value entered by user |
| `sn` | `typeId:parentId` | stock new — create a new stock record under the given parent |
| `os` | `botId` | order search — enters search mode; next user message is treated as a search query |
| `cls` | — | close/clear — dismiss the current keyboard |
| `cco` | `objId` | confirm create order — finalize a pending order creation |
| `cca` | `objId` | confirm cancel order — finalize a pending order cancellation |
| `cbc` | `objId` | confirm batch create — finalize a pending batch record creation |
| `nos` | `objId` | no-op / show — re-render the current screen without mutations (refresh) |
| `cs` | `objId:screenId` | custom screen — jump to a named screen with a specific record context |

**Interactive input flows:** Some callback ops put the chat into an input-awaiting state. The next text message from the user is intercepted (not dispatched to keyword automations) and processed as structured input:

| Trigger op | Handler | Description |
|------------|---------|-------------|
| `di` | `tryHandleDimensionsInput` | Parses dimension string (e.g. `60x40x30`) and writes to EAV fields |
| `si` | `tryHandleStockInput` | Accepts a numeric value and sets the stock field |
| `os` | `tryHandleOrderSearch` / `tryHandleClientSearch` | Searches orders or clients by the entered text, returns matching results |
| new order flow | `tryHandleNewOrderInput` | Collects order details from free-text input |

Input state is stored in Redis with a short TTL. While active, the handler intercepts the message before normal routing.

**Supported button types per screen:**
- `go` — navigate to another screen (forward)
- `_back` — linear back (restores page + filter from stack)
- `goBackTo` — named back to specific screen (clears stack above)
- `goFilter` — navigate with a filter value (e.g., status ID)
- `url` — open external URL
- `reqId` + `value` — EAV mutation (legacy, still works within screens)
- `fromValue` — conditional visibility. Button is hidden at render time if the current EAV field value does not match `fromValue`. Optional `checkReqId` specifies which requisite to check (defaults to the button's own `reqId`). Both direct and inverted EAV patterns are checked.

**Dynamic lists (listSource):**
- `queryMode`: `"table"` (root records) or `"children"` (child records)
- `typeId` — table to list, `pageSize` — items per page
- `filterReqId` — EAV field for status filtering
- `listButton.go` — navigate to detail screen on item click
- Detail screen has access to `{{id}}`, `{{val}}`, and all EAV requisites as `{{req_NNN}}`

**Template variables:** `{{screen}}` (also `{{_screen}}`), `{{_page}}`, `{{id}}`, `{{val}}`, `{{req_NNN}}`, `{{_breadcrumb}}`, `{{_filter}}`, `{{_filter_label}}`, `{{_items}}`

**Navigation stack:** Redis `tg_nav:{chatId}:{botId}` stores `{automationId, screenId, page, filterValue}`. `_back` restores full context. `goBackTo` trims stack to target screen. TTL 1 hour.

**ReplyKeyboard:**

Screens can define a `replyKeyboard` array that renders as a persistent bottom keyboard (Telegram `ReplyKeyboardMarkup`). Unlike inline keyboards (attached to a specific message), reply keyboards persist at the bottom of the chat until replaced or removed.

```json
{
  "main": {
    "text": "Choose an option",
    "replyKeyboard": [
      [{ "text": "Orders" }, { "text": "Products" }],
      [{ "text": "Settings" }]
    ]
  }
}
```

When the user taps a reply keyboard button, the bot receives it as a plain text message. The handler `tryHandleReplyKeyboardNav` scans all active `send_telegram` automations for the current bot, looking for a screen whose `replyKeyboard` contains matching button text. On match, it navigates to the corresponding screen.

Note: `editMessageText` with `InlineKeyboardMarkup` replaces the reply keyboard on that message — when transitioning away from a reply keyboard screen, send a new message rather than editing. Changing button texts in the DB does not auto-update the user's keyboard; the user must send `/menu` to refresh.

**Child items (`childItems`):**

Populates the `{{_items}}` template variable with newline-separated child record names (optionally with quantities). Configured in the `send_telegram` action:

```json
{
  "childItems": {
    "childTypeId": 123,
    "nameReqId": 456,
    "qtyReqId": 789,
    "displayReqId": 457
  }
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `childTypeId` | Yes | Type ID of child table — loads records where `up = objId AND t = childTypeId` |
| `nameReqId` | No | Requisite ID for name lookup. If the field is a ref (integer value), resolves to the referenced record's `val`. If omitted, uses the child record's own `val` |
| `qtyReqId` | No | Requisite ID for quantity. Appended as ` ×N` when value is not `"1"` |
| `displayReqId` | No | Secondary display field fetched from the referenced record (the record pointed to by `nameReqId`). Appended after the name in parentheses. Useful for showing e.g. a product's unit or category alongside its name. |

Output format: each child on a separate line, e.g. `"Молоко ×2\nХлеб\nМасло ×3"`. Empty string if no children found.

Requires `objId` in the navigation context (detail screen with a specific record).

**Child buttons (`childButtons`):**

Distinct from `childItems` (which populate `{{_items}}` text). `childButtons` render interactive toggle buttons for each child record — typically used for assembly checklists. Each child record gets a ⬜/✅ button (callback op `at`) that flips a boolean requisite on that child. A `completeButton` can gate a parent status change on all toggles being true (callback op `c`).

```json
{
  "childButtons": {
    "childTypeId": 123,
    "toggleReqId": 456,
    "completeButton": {
      "text": "Complete assembly",
      "reqId": 789,
      "value": "400",
      "fromValue": "391"
    }
  }
}
```

`completeButton.fromValue` hides the entire childButtons block when the parent's status does not match.

**Additional screen options:**

| Option | Where | Description |
|--------|-------|-------------|
| `aggregate` | screen config | When `true` on a `listSource` screen, groups list items by date and renders an "assembly day" summary — aggregates child records across all records for a given day instead of showing individual records. Used for production/assembly planning views. |
| `stockLookup` | screen config | When `true`, injects stock-related template variables (`{{_stock}}`, `{{_stockUnit}}`) for the current product record before rendering the screen text. Requires `stockTypeId` and `stockReqId` in the bot/automation config. |

See [automations.md](automations.md#send_telegram) for full action config documentation.

## Broadcast Keyboard Sync

See [portal.md](portal.md#broadcast-keyboard-sync) — when multiple recipients get the same notification with inline buttons, clicking a button syncs (removes/updates) keyboards on all other recipients' messages.

## Extended Telegram API Features

### Payments (Telegram Stars)

Support for Telegram Payments via Stars (`XTR` currency) or external payment providers.

**Webhook handlers:**
- `pre_checkout_query` → `dispatchTelegramPreCheckout()` — auto-approves unless automation explicitly rejects
- `shipping_query` → `dispatchTelegramShipping()` — for flexible pricing invoices
- `message.successful_payment` → `dispatchTelegramPayment()` — fires after successful payment

**Automation triggers:**
- `on_telegram_pre_checkout` — validate payment before charging
- `on_telegram_shipping` — provide shipping options for flexible invoices
- `on_telegram_payment` — react to successful payment (create order, update EAV, notify)

**Automation actions:**
- `send_invoice` — send invoice to recipients (supports `recipients.fromTable`)
- `answer_shipping` — respond to shipping query with options

**Template vars:** `{{_tgPaymentCurrency}}`, `{{_tgPaymentAmount}}`, `{{_tgPaymentPayload}}`, `{{_tgPaymentChargeId}}`

**AI Tools:**
- `telegram_send_invoice` (HIGH) — send invoice to chat
- `telegram_create_invoice_link` (MEDIUM) — create payment URL

### Inline Mode

Bot can respond to queries from any chat via `@botname query`.

**Webhook handler:** `inline_query` → `dispatchTelegramInlineQuery()`

**Automation trigger:** `on_telegram_inline` — receives query, must answer with results

**Automation action:** `answer_inline_query` — responds with articles/results. Supports:
- Static `results` array (InlineQueryResult objects)
- Dynamic results from table (`sourceTypeId`, `descriptionReqId`, `thumbReqId`, `messageTemplate`)

**Template vars:** `{{_tgInlineQuery}}`, `{{_tgInlineQueryId}}`, `{{_tgInlineOffset}}`

**Bot config:** Enable inline mode via BotFather. Bot must have `allowed_updates` including `inline_query`.

### Chat Administration

Bot can manage group/channel members when it's an admin.

**Webhook handler:** `chat_join_request` → `dispatchTelegramJoinRequest()`

**Automation trigger:** `on_telegram_join_request` — auto-approve/decline based on conditions

**Automation actions:**
- `telegram_ban` / `telegram_unban` — ban/unban user
- `telegram_restrict` — restrict user permissions
- `telegram_promote` — promote to admin
- `telegram_approve_join` / `telegram_decline_join` — handle join requests
- `telegram_pin` / `telegram_unpin` — pin/unpin messages
- `telegram_get_chat` — get chat info (writes to `env._tgChatInfo`)

**Template vars:** `{{_tgJoinUserId}}`, `{{_tgJoinUsername}}`, `{{_tgJoinFirstName}}`, `{{_tgJoinBio}}`, `{{_tgJoinInviteLink}}`

**AI Tools:**
- `telegram_ban_member` / `telegram_unban_member` / `telegram_restrict_member` / `telegram_promote_member` (HIGH)
- `telegram_approve_join` / `telegram_decline_join` (HIGH)
- `telegram_pin_message` / `telegram_unpin_message` (MEDIUM)
- `telegram_get_chat` / `telegram_get_chat_member_count` (LOW)
- `telegram_create_invite_link` (MEDIUM)

### Forward/Copy Messages

**Automation action:** `telegram_forward`
```json
{ "type": "telegram_forward", "botId": 1, "fromChatId": "{{_tgChatId}}", "toChatId": "-100123...", "messageId": "{{_tgMessageId}}", "copy": true }
```
- `copy: false` (default) — preserves "Forwarded from" header
- `copy: true` — sends without attribution, optionally replaces caption

**AI Tool:** `telegram_forward_message` (MEDIUM)

### Stories

Post/edit/delete stories in channels where bot is admin with `can_post_stories` permission.

**Automation action:** `telegram_post_story`
```json
{ "type": "telegram_post_story", "botId": 1, "chatId": "@channel", "content": {"type": "photo", "photo": "URL"}, "caption": "...", "activePeriod": 86400 }
```

**AI Tools:**
- `telegram_post_story` (HIGH) — publish story
- `telegram_edit_story` (HIGH) — edit story content/caption
- `telegram_delete_story` (HIGH) — delete story

### Business API

Manage business accounts connected to the bot.

**Webhook handlers:**
- `business_connection` → `dispatchTelegramBusinessConnection()` — business connected/disconnected
- `business_message` → `dispatchTelegramBusinessMessage()` — message in business chat

**Automation triggers:**
- `on_telegram_business_connection` — react to connection changes
- `on_telegram_business_message` — process business chat messages

**Automation action:** `telegram_business_reply` — send message via business_connection_id

**Template vars:** `{{_tgBusinessConnectionId}}`, `{{_tgBusinessUserId}}`, `{{_tgBusinessUsername}}`, `{{_tgBusinessChatId}}`, `{{_tgBusinessCanReply}}`, `{{_tgBusinessEnabled}}`

**AI Tools:**
- `telegram_get_business_connection` (LOW) — read connection info
- `telegram_set_business_bio` (HIGH) — set account bio
- `telegram_set_business_name` (HIGH) — set account first/last name

## Low-Level API Wrappers

All Telegram API wrappers live in `src/api/v2/utils/telegram.js`:

| Function | Telegram Method |
|----------|----------------|
| `telegramSendMessage` | sendMessage |
| `telegramSendPhoto/Video/Audio/Voice/Animation/Document` | send* media types |
| `telegramSendMediaGroup` | sendMediaGroup |
| `telegramSendLocation/Venue/Contact/Poll/Dice` | send* rich content |
| `telegramSendChatAction` | sendChatAction |
| `telegramDeleteMessage` | deleteMessage |
| `telegramForwardMessage/Messages` | forwardMessage/s |
| `telegramCopyMessage/Messages` | copyMessage/s |
| `telegramSendInvoice` | sendInvoice |
| `telegramCreateInvoiceLink` | createInvoiceLink |
| `telegramAnswerShippingQuery` | answerShippingQuery |
| `telegramAnswerPreCheckoutQuery` | answerPreCheckoutQuery |
| `telegramAnswerInlineQuery` | answerInlineQuery |
| `telegramBanChatMember` | banChatMember |
| `telegramUnbanChatMember` | unbanChatMember |
| `telegramRestrictChatMember` | restrictChatMember |
| `telegramPromoteChatMember` | promoteChatMember |
| `telegramSetChatPermissions` | setChatPermissions |
| `telegramPinChatMessage` | pinChatMessage |
| `telegramUnpinChatMessage/All` | unpinChatMessage/All |
| `telegramGetChat` | getChat |
| `telegramGetChatMember/Count/Administrators` | getChatMember* |
| `telegramCreateChatInviteLink` | createChatInviteLink |
| `telegramRevokeChatInviteLink` | revokeChatInviteLink |
| `telegramApproveChatJoinRequest` | approveChatJoinRequest |
| `telegramDeclineChatJoinRequest` | declineChatJoinRequest |
| `telegramPostStory` | postStory |
| `telegramEditStory` | editStory |
| `telegramDeleteStory` | deleteStory |
| `telegramGetBusinessConnection` | getBusinessConnection |
| `telegramReadBusinessMessage` | readBusinessMessage |
| `telegramDeleteBusinessMessages` | deleteBusinessMessages |
| `telegramSetBusinessAccountBio` | setBusinessAccountBio |
| `telegramSetBusinessAccountName` | setBusinessAccountName |
