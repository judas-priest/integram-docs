# Module: teamchat

**Path:** `src/api/v2/modules/teamchat/`
**Files:** `router.js`, `service.js`, `listeners.js`, `rooms.js`, `topics.js`, `messages.js`, `members.js`, `code-cells.js`, `ai-features.js`, `moves.js`, `standards.js`, `subscriptions.js`, `helpers.js`, `links.js`, `tc-documents.js`, `network.js`, `base-session-manager.js`, `sandbox-orchestrator.js`, `session-manager.js`, `python-session-manager.js`, `python-worker.js`, `stars.js`, `polls.js`, `receipts.js`, `snippets.js`, `room-folders.js`, `domain-roles.js`, `reactions.js`, `reminders.js`, `notification-prefs.js`, `pinned-messages.js`, `activity-digest.js`
**Base URL:** `/api/v2/:db/teamchat/...`
**Auth:** JWT required for all endpoints.

## Purpose

Teamchat provides internal workspace chat — rooms, topics, messages, and members. Designed for team discussion around knowledge-base topics.

## Endpoints

### Rooms

| Method | Path | Description |
|--------|------|-------------|
| GET | `/teamchat/rooms` | List rooms where user is member (also returns public rooms with `isMember: false`) |
| POST | `/teamchat/rooms` | Create room (auto-adds creator as admin) |
| GET | `/teamchat/rooms/:roomId` | Get room details + member/topic counts |
| PATCH | `/teamchat/rooms/:roomId` | Update room name, description, room_type, visibility |
| DELETE | `/teamchat/rooms/:roomId` | Delete room (admin only) |
| GET | `/teamchat/rooms/public` | List all public rooms with `isMember` flag (Phase 4) |
| POST | `/teamchat/rooms/:roomId/join` | Join a public room (Phase 4) |
| PATCH | `/teamchat/rooms/:roomId/folder` | Move room to folder or remove from folder (body: `{ folderId: number\|null }`) |
| POST | `/teamchat/dm/:userId` | Get or create DM room with target user |
| GET | `/teamchat/rooms/:roomId/notification-pref` | Get notification preference for current user |
| PUT | `/teamchat/rooms/:roomId/notification-pref` | Set notification preference (body: `{ level: 'all'\|'mentions'\|'nothing' }`) |

**MCP tool gaps:** `create_room` only supports `name` and `isPublic` (REST accepts 7 fields: name, description, room_type, linked_to_type, linked_to_id, visibility, auto_agent). `update_room` only supports `name` (REST also supports description, room_type, visibility).

### Room Folders

| Method | Path | Description |
|--------|------|-------------|
| GET | `/teamchat/folders` | List all room folders with room counts |
| POST | `/teamchat/folders` | Create room folder (body: `{ name, parentId?, icon?, sortOrder? }`) |
| PATCH | `/teamchat/folders/:folderId` | Update room folder (name, parentId, icon, sortOrder) |
| DELETE | `/teamchat/folders/:folderId` | Delete room folder (unlinks rooms, does not delete rooms) |

### Topics

| Method | Path | Description |
|--------|------|-------------|
| GET | `/teamchat/topics` | Cross-room topic search (cursor pagination) |
| POST | `/teamchat/rooms/:roomId/topics` | Create topic and link to room. Body accepts optional `author_type` (server enforces `'human'` for non-agent users) |
| GET | `/teamchat/rooms/:roomId/topics` | List topics in a room |
| POST | `/teamchat/rooms/:roomId/topics/:topicId` | Attach existing topic to room |
| DELETE | `/teamchat/rooms/:roomId/topics/:topicId` | Detach topic from room |
| GET | `/teamchat/topics/:topicId` | Get topic details + which rooms contain it |
| PATCH | `/teamchat/topics/:topicId` | Update topic fields (name, status, pinned, assigned_to, deadline_at, priority) |
| PATCH | `/teamchat/topics/:topicId/status` | Update topic status (open, in_progress, resolved, closed) |
| POST | `/teamchat/topics/:topicId/pin` | Pin a topic (Phase 5) |
| DELETE | `/teamchat/topics/:topicId/pin` | Unpin a topic (Phase 5) |
| POST | `/teamchat/topics/:topicId/task` | Convert topic to task — set assigned_to, deadline_at, priority (Phase 5) |
| DELETE | `/teamchat/topics/:topicId/task` | Revert task back to discussion — clear assigned_to/deadline_at (Phase 5) |
| POST | `/teamchat/topics/:topicId/summarize` | Generate AI summary of topic discussion (optional `since`: last_summary, 1h, 4h, 24h, 7d, all) |
| POST | `/teamchat/topics/:topicId/move` | Move topic to a different room (admin only) |
| DELETE | `/teamchat/topics/:topicId` | Delete topic (admin only, cascades messages) |
| GET | `/teamchat/topics/:topicId/link` | Get shareable deep link to topic (Phase 4) |
| POST | `/teamchat/topics/:topicId/export-to-document` | Export topic messages to a workspace document (Phase 4) |
| POST | `/teamchat/topics/:topicId/refresh-document` | Refresh exported document with new messages (Phase 4) |

### Messages

| Method | Path | Description |
|--------|------|-------------|
| GET | `/teamchat/topics/:topicId/messages` | List messages (cursor pagination) |
| POST | `/teamchat/topics/:topicId/messages` | Create message. Body accepts optional `author_type` (server enforces `'human'` for non-agent users) |
| PATCH | `/teamchat/messages/:msgId` | Edit own message text (sets edited_at) |
| DELETE | `/teamchat/messages/:msgId` | Soft-delete message (own or admin) |
| POST | `/teamchat/messages/:msgId/move` | Move single message to a different topic (author within 72h, or room admin) |
| POST | `/teamchat/messages/move-bulk` | Move multiple messages to a different topic (room admin only) |
| GET | `/teamchat/messages/search` | Full-text search across messages. Query params: `q` (search text), `room_id` (filter by room), `author` (filter by author), `author_type` (filter by author type), `limit` (1-200) |
| POST | `/teamchat/messages/:msgId/forward` | Forward message to another topic (body: `{ target_topic_id }`) |
| GET | `/teamchat/messages/:msgId/link` | Get shareable deep link to message (Phase 4) |

### Pinned Messages

| Method | Path | Description |
|--------|------|-------------|
| POST | `/teamchat/topics/:topicId/messages/:msgId/pin` | Pin message |
| DELETE | `/teamchat/topics/:topicId/messages/:msgId/pin` | Unpin message |
| GET | `/teamchat/topics/:topicId/pinned` | List pinned messages |

### Members

| Method | Path | Description |
|--------|------|-------------|
| GET | `/teamchat/rooms/:roomId/members` | List room members |
| POST | `/teamchat/rooms/:roomId/members` | Add member (upsert) |
| DELETE | `/teamchat/rooms/:roomId/members/:userId` | Remove member |
| PATCH | `/teamchat/rooms/:roomId/members/:userId` | Update member role (admin/member) |

### Domain Roles

| Method | Path | Description |
|--------|------|-------------|
| GET | `/teamchat/rooms/:roomId/roles` | List domain roles for a room |
| POST | `/teamchat/rooms/:roomId/roles` | Create domain role (room admin only) |
| PATCH | `/teamchat/rooms/:roomId/roles/:roleId` | Update domain role (room admin only) |
| DELETE | `/teamchat/rooms/:roomId/roles/:roleId` | Delete domain role (room admin only) |
| PUT | `/teamchat/rooms/:roomId/members/:userId/domain-role` | Assign domain role to member (room admin only) |

### File Upload

| Method | Path | Description |
|--------|------|-------------|
| POST | `/teamchat/topics/:topicId/upload` | Upload file and create message with file card (multipart, max 50 MB) |

### Approvals

| Method | Path | Description |
|--------|------|-------------|
| POST | `/teamchat/topics/:topicId/approvals/:actionId` | Approve or reject a pending agent action (admin only) |

### Recent Topics + Muting

| Method | Path | Description |
|--------|------|-------------|
| GET | `/teamchat/topics/recent` | List recent topics across all rooms (cursor pagination, filter: all/participated/unread/resolved/assigned_to_me/overdue/pinned) |
| POST | `/teamchat/topics/:topicId/mute` | Mute a topic (hide from recent list, skip WS notifications) |
| DELETE | `/teamchat/topics/:topicId/mute` | Unmute a topic |
| GET | `/teamchat/topics/muted` | List all muted topics for current user |
| POST | `/teamchat/topics/:topicId/mark-read` | Mark topic as read for current user |
| GET | `/teamchat/topics/tasks` | List tasks (topics with assigned_to) with sort: priority/recent/deadline (Phase 5) |
| GET | `/teamchat/topics/kanban` | Kanban board grouped by status, optional room_id filter (Phase 5) |

### Notebook Cells — Reorder

| Method | Path | Description |
|--------|------|-------------|
| PATCH | `/teamchat/topics/:topicId/cell-order` | Reorder notebook cells (`ids`: array of message IDs in desired order) |

### Subscriptions

| Method | Path | Description |
|--------|------|-------------|
| POST | `/teamchat/topics/:topicId/subscribe` | Subscribe topic to table/record changes (`pointType`: table\|record, `pointId`) |
| DELETE | `/teamchat/topics/:topicId/subscribe` | Unsubscribe from table/record changes (`pointType`, `pointId`) |
| GET | `/teamchat/topics/:topicId/subscriptions` | List active subscriptions for a topic |

### Starred Messages

| Method | Path | Description |
|--------|------|-------------|
| POST | `/teamchat/messages/:msgId/star` | Star (bookmark) a message |
| DELETE | `/teamchat/messages/:msgId/star` | Unstar a message |
| GET | `/teamchat/messages/starred` | List starred messages for current user (cursor pagination) |

### Topic Follow

| Method | Path | Description |
|--------|------|-------------|
| POST | `/teamchat/topics/:topicId/follow` | Follow a topic (get notifications) |
| DELETE | `/teamchat/topics/:topicId/follow` | Unfollow a topic |
| GET | `/teamchat/topics/followed` | List followed topics for current user |

### Message Edit History

| Method | Path | Description |
|--------|------|-------------|
| GET | `/teamchat/messages/:msgId/history` | Get edit history for a message (list of old text versions) |

Edit history is saved automatically when a message is edited via `PATCH /messages/:msgId`.

### Polls

| Method | Path | Description |
|--------|------|-------------|
| POST | `/teamchat/topics/:topicId/polls` | Create poll (body: `{ question, options[], multi?, anonymous? }`) — creates message + poll record |
| GET | `/teamchat/polls/:pollId` | Get poll with vote counts, voters (if not anonymous), user's votes |
| POST | `/teamchat/polls/:pollId/vote` | Vote on option (body: `{ optionIdx }`) — single-choice clears prior votes |
| DELETE | `/teamchat/polls/:pollId/vote` | Remove vote (body: `{ optionIdx }`) |
| PATCH | `/teamchat/polls/:pollId/close` | Close poll (creator only) |
| POST | `/teamchat/polls/:pollId/options` | Add option (body: `{ text }`) — anyone can add options |

### Read Receipts

| Method | Path | Description |
|--------|------|-------------|
| POST | `/teamchat/topics/:topicId/receipt` | Update read receipt (body: `{ lastReadId }`) — upsert with GREATEST |
| GET | `/teamchat/topics/:topicId/receipts` | Get all read receipts for a topic |
| GET | `/teamchat/messages/:msgId/readers` | Get users who have read this message |
| POST | `/teamchat/topics/:topicId/mark-unread` | Mark topic as unread (body: `{ fromMessageId? }`) — resets receipt or deletes it entirely |

### Reminders

| Method | Path | Description |
|--------|------|-------------|
| GET | `/teamchat/reminders` | List pending reminders for current user |
| POST | `/teamchat/topics/:topicId/reminders` | Create reminder (body: `{ fireAt: ISO string, messageId?, note? }`) |
| DELETE | `/teamchat/reminders/:reminderId` | Cancel a pending reminder (owner only) |

### Saved Snippets

| Method | Path | Description |
|--------|------|-------------|
| GET | `/teamchat/snippets` | List own snippets + shared snippets |
| POST | `/teamchat/snippets` | Create snippet (body: `{ title, content, shortcut?, shared? }`) |
| PATCH | `/teamchat/snippets/:snippetId` | Update snippet (owner only) |
| DELETE | `/teamchat/snippets/:snippetId` | Delete snippet (owner only) |

### Message Reactions

| Method | Path | Description |
|--------|------|-------------|
| GET | `/teamchat/messages/:msgId/reactions` | Get reactions for a message (grouped by emoji: `[{emoji, count, authors}]`) |
| POST | `/teamchat/messages/:msgId/reactions` | Add reaction (body: `{ emoji }`) — idempotent (ON CONFLICT DO NOTHING) |
| DELETE | `/teamchat/messages/:msgId/reactions/:emoji` | Remove own reaction for the given emoji |

## Data Model

```
Room {
  id, name, description, room_type (group|direct|support),
  visibility (private|public, default private),
  history_mode (shared, default shared),
  auto_agent (TEXT, agent slug for auto-respond),
  linked_to_type, linked_to_id (for linking to EAV objects),
  folder_id (BIGINT, nullable — references RoomFolder),
  created_by, created_at, updated_at
}

RoomFolder {
  id, name, parent_id (nullable, for nested folders), icon (VARCHAR(64)),
  sort_order (INT, default 0), created_by, created_at, updated_at
}

RoomRole {
  id BIGINT IDENTITY PK,
  room_id BIGINT FK rooms CASCADE,
  name VARCHAR(64),
  agent_slugs TEXT[] DEFAULT '{}',
  created_at TIMESTAMPTZ,
  UNIQUE(room_id, name)
}

Topic {
  id, name, status (open|in_progress|resolved|closed, default open),
  mode (TEXT, e.g. 'sandbox' — controls agent behavior for the topic; currently read-only — no REST or MCP path to set/update it after creation),
  pinned (boolean, default false), assigned_to (varchar), deadline_at (timestamptz),
  priority (high|medium|low, default medium), linked_doc_id,
  created_by, created_at, updated_at
}

// is_task = assigned_to IS NOT NULL (virtual, not a column)

RoomTopic (junction) {
  room_id, topic_id, added_by, added_at
}

Message {
  id, topic_id, author, author_type (human|bot|agent),
  text, reply_to_id, mentions (JSONB), file_ids (JSONB),
  task_action (JSONB), agent_metrics (JSONB), cards (JSONB),
  cell_order (INT, ordering for notebook cells within a topic),
  embedding (VECTOR(1024)), edited_at,
  deleted (soft delete), created_at
}

Member {
  room_id, user_id, role (admin|member),
  domain_role_id BIGINT FK room_roles,
  joined_at, last_read_at
}

TopicMute {
  user_id, topic_id (FK to topics, CASCADE), muted_at (PK: user_id, topic_id)
}

TopicFollow {
  user_id, topic_id (FK to topics, CASCADE), created_at (PK: user_id, topic_id)
}

Star {
  user_id, message_id (FK to messages, CASCADE), created_at (PK: user_id, message_id)
}

EditHistory {
  id, message_id (FK to messages, CASCADE), old_text, edited_by, edited_at
}

Poll {
  id, message_id (FK to messages, CASCADE, UNIQUE),
  question, options (JSONB array), multi (bool), anonymous (bool), closed (bool),
  created_by, created_at
}

PollVote {
  id, poll_id (FK to polls, CASCADE), option_idx, user_id, voted_at
  UNIQUE(poll_id, option_idx, user_id)
}

Receipt {
  user_id, topic_id (FK to topics, CASCADE), last_read_id, read_at
  PK(user_id, topic_id)
}

Snippet {
  id, title, content, shortcut, created_by, shared (bool), created_at, updated_at
}

Reminder {
  id, user_id, topic_id (FK to topics, CASCADE),
  message_id (FK to messages, SET NULL), note,
  fire_at (timestamptz), status (pending|fired|cancelled, default pending),
  job_id (BullMQ job ID for cancellation), created_at
}

MessageReaction {
  id, message_id (FK to messages, CASCADE),
  author, emoji,
  created_at
  UNIQUE(message_id, author, emoji)
}

NotificationPref {
  room_id, user_id, level (all|mentions|nothing)
  PK(room_id, user_id)
}

PinnedMessage {
  topic_id, message_id, pinned_by, pinned_at
  UNIQUE(topic_id, message_id)
}
```

## Events Emitted

- `teamchat.room.created` — `{ db, pool, roomId, name, createdBy }`
- `teamchat.room.deleted` — `{ db, pool, roomId }`
- `teamchat.topic.created` — `{ db, pool, roomId, topicId, name, createdBy }`
- `teamchat.topic.deleted` — `{ db, pool, topicId }`
- `teamchat.topic.moved` — `{ db, pool, topicId, sourceRoomId, newRoomId, movedBy }`
- `teamchat.message.created` — `{ db, pool, topicId, messageId, text, author, authorType, replyToId, mentions, agentMetrics, createdAt }`
- `teamchat.message.moved` — `{ db, pool, msgId, sourceTopicId, targetTopicId, movedBy }` (single) or `{ db, pool, msgIds, sourceTopicIds, targetTopicId, movedBy }` (bulk)
- `teamchat.topic.summarized` — `{ db, pool, topicId, messageCount, generatedBy }`
- `teamchat.message.updated` — `{ db, pool, topicId, messageId, text, editedAt, author }`
- `teamchat.message.deleted` — `{ db, pool, topicId, messageId }`
- `teamchat.member.added` — `{ db, pool, roomId, userId, role, addedBy }`
- `teamchat.member.removed` — `{ db, pool, roomId, userId, removedBy }`
- `teamchat.debate.started` — `{ db, pool, topicId, proposition, agentId, key }` — a debate was started by an agent
- `teamchat.debate.resolved` — `{ db, pool, topicId, key, debate }` — both agents responded, debate marked as resolved
- `teamchat.topic.muted` — `{ db, pool, topicId, userId }` — a topic was muted by a user
- `teamchat.topic.unmuted` — `{ db, pool, topicId, userId }` — a topic was unmuted by a user
- `teamchat.room.joined` — `{ db, pool, roomId, userId }` — a user joined a public room (Phase 4)
- `teamchat.topic.exported` — `{ db, pool, topicId, documentId }` — a topic was exported to a document (Phase 4)
- `teamchat.message.cards_updated` — `{ db, pool, topicId, messageId, cards }` — message cards changed (e.g. code cell status update). Broadcasts `tc:message-updated` via WS with the updated `cards` payload.

**No bus events yet:** reactions, stars, edit history, polls, receipts have no bus events — they use direct WS broadcast or no broadcast at all.

## Agent Behaviors

### Agent Response Loop

When an agent is a room member, it actively listens and responds:

1. **@-mention** — any message mentioning `@agent:<name>` triggers an agent response via `respondToMessage()`. The agent analyzes the message context, searches relevant decisions and discussions, and posts a reply in the same topic.
2. **Autonomous triggers** — certain keywords (`найди`, `похожие`, `решения`, `крыло`) with agent mentions trigger autonomous search and recommendation flows.
3. **Behavioral constraints** — agents cannot delete messages, change verdicts on decisions, or write to personal rooms. All agent actions are visible to the team.

### Topic Summarization

Users can generate AI summaries of topic discussions via `POST /topics/:topicId/summarize` or the `summarize_topic` MCP tool. The feature:

1. Loads up to 200 messages (filtered by optional `since` parameter: `last_summary`, `1h`, `4h`, `24h`, `7d`, `all`)
2. Calls LLM with a system prompt to extract key conclusions, decisions, open questions, and positions
3. Inserts a summary message with `author='agent:system'`, `author_type='agent'`, `agent_metrics.type='summary'`
4. Stores the summary in `agent_memory` (public table, key `teamchat:summary:{topicId}`) via sideEffect
5. Emits `teamchat.message.created` (for WS broadcast) and `teamchat.topic.summarized`

Summary messages render with a blue left border and a "Саммари" badge in the frontend.

### Periodic Consolidation

Two mechanisms keep agent context fresh:

- **Micro-consolidation** — every 50 messages in a room, the system triggers `consolidateContext()` for all agent members. The LLM generates a 3-5 sentence summary covering key decisions, conflicts, and patterns. Stored to `agent_memory` with tags `['summary', 'room:{roomId}', 'teamchat']` and posted as a context message in the chat.
- **On-resolve consolidation** — `teamchat.topic.resolved` triggers the same flow immediately for all agents in the room.

### Debate Pattern

Debates are structured multi-agent discussions where two agents argue opposing positions:

1. **Initiation** — an agent proposes a proposition (e.g., `agent:analyst` suggests "использовать углепластик для крыла"). `teamchat.debate.started` event is emitted.
2. **Pairing** — `teamchat.debate.started` notifies `agent:reviewer` and `agent:analyst`. Each agent is assigned a position (proponent/opponent).
3. **Response loop** — `teamchat.message.created` in active debate topics auto-responds via `respondToDebate()`. Each agent gets one response. After both respond, the debate is marked as resolved via `teamchat.debate.resolved`.
4. **Resolution** — `debate` object (both positions) is stored, and the result is posted to the topic. The initiating agent may summarize.

### Delegation Chain Limits

Agent-to-agent delegation (when one agent @-mentions another) is rate-limited to prevent runaway loops:

- `MAX_TOTAL_DELEGATIONS = 5` — maximum agent-authored messages allowed in a single topic within the timeout window
- `CHAIN_TIMEOUT_MS = 15 minutes` — sliding window for counting delegations
- Counts ALL `author LIKE 'agent:%'` messages in the topic within the window (does not filter by mentions, since `handleAgentResponseLoop` creates messages without passing `mentions`)
- Fails closed — on error checking limits, delegation is blocked

Defined in `listeners.js:checkDelegationChain()`.

### Standards-Based Code Review (`standards.js`)

Supports auto code review by searching for applicable ADRs (via typed links and semantic search on the commit diff) and loading the repo's `STANDARDS.md` file. Used by the commit detection listener to provide context to the reviewer agent.

- `getApplicableStandards(pool, db, repo, diff, roomId)` — returns `{ adrs, repoStandards }`
- `formatStandardsForPrompt(standards)` — formats for LLM prompt injection

### Data Card Enrichment (`helpers.js:enrichDataCards`)

Messages referencing `@record TypeID:ObjectID`, `@doc DocID`, or `@report ReportID` are auto-enriched with inline data cards. The function parses these references from message text, loads the referenced data, merges it into the message's `cards` JSONB column, and broadcasts the update via WS.

### Periodic Consolidation (6-hour interval)

In addition to micro-consolidation (every 50 messages), a 6-hour periodic consolidation runs for active rooms. Room activity is tracked in an in-memory `Map` (`_roomActivity`, max 200 rooms). When a room has had activity and the interval has elapsed, all agent members in that room get their context consolidated via `consolidateContext()`.

## Listeners (`listeners.js`)

Registered at startup. Listens to bus events for:

- **Embedding** — `teamchat.message.created` triggers async embedding via `embedText()`, storing the vector in `_v2_tc_messages.embedding`
- **WebSocket broadcasting** (single-process only — uses in-memory `Map` in ws.js, no Redis pub/sub; does not scale to multiple backend instances) — relays events to WS clients subscribed to `tc:room:<roomId>`:
  - `tc:new-message` — new message in a room
  - `tc:message-updated` — message text edited
  - `tc:message-deleted` — message soft-deleted
  - `tc:topic-created` — new topic linked to a room
  - `tc:user-joined` — member added
  - `tc:user-left` — member removed
  - `tc:typing` — typing indicator (relayed client→server, see ws.js)
  - `tc:cell-log` — streaming console output from notebook cell execution (`{ msgId, line }`)
  - `tc:topic-deleted` — topic was deleted
  - `tc:agent-thinking` — agent is processing a response (thinking indicator)
- **Agent onboarding** — `teamchat.member.added` for agent users (`agent:*`) posts a history summary of the last 20 topics in the room
- **Context consolidation** — `teamchat.topic.resolved` triggers consolidation for all agents in the room via `consolidateContext()`. Generate LLM summarization (3-5 sentences covering key decisions, conflicts, and patterns), store to `agent_memory` with tags `['summary', 'room:{roomId}', 'teamchat']`, post context message to chat.
- **Micro-consolidation** — every 50 messages per room triggers consolidation for all agent members (same LLM summarization flow)
- **Code commit detection** — `teamchat.message.created` with commit keywords (`закоммитил`, `commit_file`, etc.) pings `@reviewer-agent`
- **Debate handling** — `teamchat.debate.started` notifies analysts and reviewers; `teamchat.message.created` in active debate topics auto-responds via `respondToDebate()`
- **Auto code review on agent commits** — `codespace.commit.created` with `authorName` starting with `agent:` triggers:
  1. Fetches the commit diff via `getCommitDiff()`
  2. Finds the teamchat room linked to the repo (`linked_to_type='codespace'`) or by repo slug name
  3. Creates topic `Code Review: {branch}` in that room
  4. Ensures the `agent:reviewer-agent` is a member of the room
  5. Posts the diff in a message mentioning `@reviewer-agent`

## Access Control

All room-scoped endpoints require the caller to be a **member** of the room (`_v2_tc_members`). Non-members receive HTTP 403.

| Function | Check |
|---|---|
| `getRoom`, `listTopics`, `listMembers` | `ensureMember` — user must be in `_v2_tc_members` for the room |
| `updateRoom` | `ensureAdmin` — user must be a room admin. Changing `visibility` additionally requires workspace admin role. |
| `attachTopic`, `detachTopic` | `ensureMember` — user must be a room member |
| `createTopic` | `ensureMember` — user must be a room member (skipped for agent/system users) |
| `getTopic`, `updateTopic`, `updateTopicStatus` | `ensureTopicMember` — user must be a member of at least one room containing the topic |
| `pinTopic`, `unpinTopic`, `makeTask`, `unmakeTask` | `ensureTopicMember` — user must be a member of at least one room containing the topic |
| `listAllTopics` | Filtered by membership — only topics from rooms the user belongs to |
| `listMessages`, `createMessage`, `summarizeTopic` | Topic-access check — user must be a member of at least one room containing the topic |
| `searchMessages` with `room_id` | `ensureMember` on the given room; without `room_id` — no membership check (cross-room agent search) |
| `addMember`, `removeMember`, `updateMemberRole` | `ensureMember` — caller must be a room member |
| `deleteRoom` | `ensureAdmin` — caller must have `role='admin'` in `_v2_tc_members` for this room (not workspace role) |
| `deleteTopic` | Room-admin check via JOIN — caller must be `role='admin'` in at least one room containing the topic |
| `moveTopic` | `ensureAdmin` — caller must have `role='admin'` in the source room |
| `moveMessage` | Author (within 72h) or room-admin check via JOIN — caller must be the message author or a room admin |
| `moveMessagesBulk` | Room-admin check per source topic — caller must be admin of every room containing the message source topics |

### Last-admin safeguard

`removeMember` and `updateMemberRole` (when demoting to `member`) prevent removing or demoting the **last admin** in a room. The server checks `COUNT(*)` of remaining admins before allowing the operation.

### Analytics

| Method | Path | Description |
|--------|------|-------------|
| GET | `/teamchat/analytics/wmatrix` | W-matrix (ONA graph): collaboration graph between team members. Query params: `roomId` (optional filter), `days` (1-365, default 30). Returns `{ nodes, edges, metrics: { hubs, bridges, isolated } }` |

## Collaborative Sandbox (Code Cells)

Topics support executable code cells embedded in messages. A message with `cards` containing an entry of `type='code_cell'` is rendered as an interactive code editor in the frontend.

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/teamchat/topics/:topicId/cells/:msgId/run` | Execute code cell — JS via V8 isolate, Python via server-side Pyodide worker thread. Returns `{ output: { text, images, tables, error, duration_ms } }` |
| POST | `/teamchat/topics/:topicId/cells/:msgId/accept` | Commit the cell's code to the codespace repo for this topic; returns `{ commitSha, path }` |
| POST | `/teamchat/topics/:topicId/cells/:msgId/publish` | Publish accepted JS cell as portal custom_code widget (Vue SFC wrapper → codespace `custom_code/` dir). Only JS cells with status `accepted`. Status changes to `published`. |
| POST | `/teamchat/topics/:topicId/cells/:msgId/output` | Save output from a Python cell; accepts `{ output: { text, images, tables, error, duration_ms } }` |
| PUT | `/teamchat/topics/:topicId/cells/:msgId/yjs-state` | Persist Yjs CRDT document state (base64 binary). Auto-called every 30s + on tab close. Checks room membership. |

### Session Manager

**JavaScript:** Each topic gets one persistent `isolated-vm` isolate, created on first `run` and kept alive for 10 minutes of inactivity (TTL resets on each run). Limits: max 5 active sessions per workspace, 128 MB memory per isolate. On TTL expiry the isolate is destroyed; the next `run` creates a fresh one.

**Python:** Each topic gets one persistent Pyodide worker thread (`python-session-manager.js`), created on first Python cell `run`. Same TTL (10 min). Limits: max 2 Python sessions total (~200MB each). Pyodide loads numpy, pandas, matplotlib via micropip. Variables persist across cells (shared state). Matplotlib captures → base64 PNG images, DataFrame captures → JSON tables.

### Sandbox Agent Cycle (Phase 3b)

When a topic has `mode='sandbox'`, human messages auto-trigger an agent chain:
1. **Researcher** (`sandbox-researcher`) — RAG context via kag_search, search_similar_decisions, list_tables/list_objects
2. **Engineer** (`sandbox-engineer`) — generates code cells via `send_teamchat_message` with `cards.type='code_cell'` (note: `send_teamchat_message` is defined only in `sandbox-engineer.js`, not in global TOOL_DEFS — only the sandbox-engineer agent has access to it)
3. **Opponent** (`sandbox-opponent`) — reviews code, finds bugs
4. **Controller** (`sandbox-controller`) — informed check: does the code match the user's task and project standards
5. **Blind review** (issue #44) — the opponent re-judges the engineer's code cells WITHOUT the task/spec (reasons backward from the implementation; no tools, no chat history). Deterministic spec-independent red flags (`empty-catch`, `unconditional-success`, `dead-branch`, `free-variable`) are scanned even if the LLM pass fails. Findings the blind pass catches beyond the informed reviewers are marked `by: opponent-blind` in the verdict message — a high-value signal. Logic lives in `agents/sandbox-opponent.js` (`blindSystemPrompt`, `scanRedFlags`, `diffFindings`).

Orchestrator: `sandbox-orchestrator.js`. Mutex prevents parallel cycles. 5-minute total timeout. Uses `runner.run()` directly (NOT agent-registry invoke). Agent definitions auto-loaded by `AgentCatalog` from `agents/` directory.

### Streaming Console Output (Phase 2)

During JavaScript execution the isolate emits `console.log` calls as WebSocket events (`tc:cell-log`) with `{ msgId, line }` — sent to the topic's room subscribers before the final response arrives. This allows the frontend to stream output live for long-running cells.

### Python Execution via Pyodide (Phase 2)

Python cells (`language: 'python'` in the `code_cell` card) run server-side via a dedicated worker thread (`python-worker.js`) managed by `python-session-manager.js`. The worker executes the code using Pyodide, captures stdout, and returns the result. Output is persisted via `POST /topics/:topicId/cells/:msgId/output`.

### Rich Output Rendering (Phase 2)

Cell output supports two rich formats beyond plain text:

- **matplotlib PNG** — stored in `output.images[]` as base64. Rendered as an `<img>` in `CodeCellMessage.vue`.
- **DataFrame table** — stored in `output.tables[]` as `{ columns, rows }`. Rendered as a PrimeVue `DataTable` component. Never uses `v-html`.

Plain text output continues to use `v-text`.

### Bridges Available in Cells

All standard sandbox bridges are available: `query`, `getRecord`, `updateRecord`, `createRecord`, `deleteRecord`, `ai`, `fetch`. The `fetch` bridge enforces `closed_contour` — external URLs not on the workspace allowlist are blocked (same enforcement as the `browse` bridge). Additionally, two teamchat-specific bridges are injected:

- `kag_search(query, topK?)` — semantic search over the knowledge-attribute graph
- `search_decisions(query, topK?)` — vector search over archived decisions in the topic's rooms

### Commit Metadata (Phase 2)

`POST /cells/:msgId/accept` returns and stores enriched commit metadata beyond `{ commitSha, path }`:

- `source` — origin label (e.g. `teamchat-sandbox`)
- `topic` — topic name at commit time
- `agent` — agent slug if the cell was authored by an agent (`author_type='agent'`)
- `kagRefs` — array of KAG node IDs referenced in the cell output (populated from the last `kag_search` call in the session)

### Frontend

Code cells are rendered by `CodeCellMessage.vue` using a CodeMirror 6 editor (JavaScript mode, dark theme). Cell language is controlled by the `language` field in the `code_cell` card (`'javascript'` default, `'python'` for Pyodide cells). Run and Accept buttons call `sandboxService.runCell()` and `sandboxService.acceptCell()` respectively.

## Inline Cards

Messages that reference `@decision #N` are auto-enriched with a `cards` JSONB field containing decision details and conflicts. Enrichment runs as a `sideEffect` in `teamchat.message.created` listener. The `cards` field is included in `listMessages` and `createMessage` responses, and broadcast via `tc:message-updated` WebSocket event.

## Meta KB Auto-Agent

Rooms can have an `auto_agent` column (TEXT) on `_v2_tc_rooms`. When set to an agent slug (e.g. `'teamchat-agent'`), every new human message in any topic of that room triggers an automatic agent response via `respondToMessage()`.

`createMessage` accepts an optional `skipAgentResponse: true` flag (internal use). When set, the auto-agent listener does not fire — used by the agent itself when posting its own reply to avoid infinite loops.

The `meta-kb` room uses `auto_agent = 'teamchat-agent'` to power the Meta KB chat mode in `AiPanel`.

## DB Tables (per-workspace, schema `"db"`)

- `_v2_tc_rooms` — chat rooms (`auto_agent` TEXT column — slug of agent to auto-respond in this room; `folder_id` BIGINT nullable — references `_v2_tc_room_folders`)
- `_v2_tc_room_folders` — room folder tree (id, name, parent_id, icon, sort_order, created_by, timestamps)
- `_v2_tc_room_roles` — domain roles per room (id, room_id FK CASCADE, name, agent_slugs TEXT[], created_at; UNIQUE(room_id, name))
- `_v2_tc_topics` — discussion topics (global, cross-room via junction)
- `_v2_tc_room_topics` — room↔topic junction (CASCADE delete on both FKs)
- `_v2_tc_messages` — messages with optional vector embeddings
- `_v2_tc_members` — room membership (PK: room_id, user_id)
- `_v2_tc_topic_mutes` — per-user topic muting (PK: user_id, topic_id, CASCADE on topic delete)
- `_v2_tc_subscriptions` — data subscriptions linking topics to table/record change events (columns: id BIGINT IDENTITY, topic_id BIGINT FK to topics CASCADE, point_type TEXT, point_id BIGINT, created_by VARCHAR(128), created_at TIMESTAMPTZ; UNIQUE(topic_id, point_type, point_id))
- `_v2_tc_stars` — starred/bookmarked messages (PK: user_id, message_id; CASCADE on message delete)
- `_v2_tc_follows` — followed topics (PK: user_id, topic_id; CASCADE on topic delete)
- `_v2_tc_edit_history` — message edit history (id, message_id FK CASCADE, old_text, edited_by, edited_at; index on message_id)
- `_v2_tc_polls` — polls attached to messages (id, message_id FK CASCADE UNIQUE, question, options JSONB, multi, anonymous, closed, created_by, created_at)
- `_v2_tc_poll_votes` — poll votes (id, poll_id FK CASCADE, option_idx, user_id, voted_at; UNIQUE(poll_id, option_idx, user_id))
- `_v2_tc_receipts` — read receipts per topic (PK: user_id, topic_id; last_read_id, read_at)
- `_v2_tc_snippets` — saved text snippets (id, title, content, shortcut, created_by, shared, created_at, updated_at)
- `_v2_tc_reminders` — message reminders (id, user_id, topic_id FK CASCADE, message_id FK SET NULL, note, fire_at, status pending|fired|cancelled, job_id, created_at; index on user_id+status+fire_at)
- `_v2_tc_message_reactions` — message reactions (id, message_id FK CASCADE, author, emoji, created_at; UNIQUE(message_id, author, emoji))
- `_v2_tc_notification_prefs` — per-room notification preferences (room_id, user_id, level; PK(room_id, user_id))
- `_v2_tc_pinned_messages` — pinned messages per topic (topic_id, message_id, pinned_by, pinned_at; UNIQUE(topic_id, message_id))

## Indexes

- GIN `to_tsvector('russian', text)` on messages for full-text search
- HNSW `(embedding vector_cosine_ops)` on messages for vector similarity
- B-tree on all FK columns
- Partial indexes on topics: `idx_tc_topics_pinned` (WHERE pinned), `idx_tc_topics_assigned` (WHERE assigned_to IS NOT NULL), `idx_tc_topics_deadline` (WHERE deadline_at IS NOT NULL), `idx_tc_topics_status`
- `idx_tc_rooms_visibility` on rooms
- `idx_tc_messages_author` on messages (author) WHERE deleted = false — speeds up agent delegation chain checks
- `idx_tc_rooms_linked` on rooms (linked_to_type, linked_to_id) — speeds up finding rooms linked to EAV objects/codespaces

## Activity Intelligence

### Service (`activity-digest.js`)

`activity-digest.js` exports `getTeamActivity(pool, db, options)` — aggregates team and user activity data from teamchat messages, topics, and rooms.

### AI Tools

| Tool | Risk Tier | Description |
|------|-----------|-------------|
| `get_team_activity` | TIER_LOW | Get aggregated team activity across rooms and topics |
| `get_user_activity` | TIER_LOW | Get activity summary for a specific user |
| `generate_team_digest` | TIER_LOW | Generate an AI-powered team activity digest |

Tool implementations are in `ai/agent/tools/activity.js`.

### Dedicated Agent

The `activity-agent` (slug `activity-agent`) is registered in the agent catalog and handles activity-related queries using the tools above.

## Portal Proxy (`/portal/api/teamchat/*`)

**File:** `portal/teamchat/router.js`
**Endpoints:** 62+ endpoints proxied from teamchat service
**Auth:** `portal_jwt` → synthetic user `portal:<clientObjectId>`
**Rate limit:** 60 req/min per portal client

### Access Control

- Room must be in `config.teamchat.roomIds` OR `room_type='support'` OR `visibility='public'`
- Identity: `portal/teamchat/identity.js` — `buildPortalUser(clientObjectId)`
- Access: `portal/teamchat/access.js` — `isRoomAccessible()`, `filterAccessibleRooms()`

### Portal-Specific Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/rooms/:roomId/agents` | Agents in room enriched with registry metadata (slug, name, description, category, capabilities, model) |

### Message Field Names (via listMessages)

- `{id, author, authorType, text, replyToId, cards, createdAt}`
- `author` not `createdBy`, `replyToId` not `reply_to_id`
- `createMessage` body accepts `reply_to_id` (snake_case)

### Pinned Messages (via listPinnedMessages)

- Returns `{messageId, text, author, pinnedBy, pinnedAt, createdAt}`
- Note: `messageId` not `id`

### Search (via searchMessages)

- `GET /messages/search?q=...&room_id=...`
- Accepts `room_id` NOT `topicId`
