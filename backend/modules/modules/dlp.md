# Module: dlp

**Path:** `src/api/v2/modules/ai/` (DLP endpoints in `router.js`, engine in `contour-guard.js`)
**Files:** `router.js` (DLP CRUD section), `contour-guard.js`
**Base URL:** `/api/v2/:db/ai/dlp/...`
**Auth:** JWT required. All endpoints require `admin` role.

## Purpose

DLP (Data Loss Prevention) contour guard — prevents sensitive data leakage to external LLM providers. Called from `llm-router.js` before sending requests to external models. Only active when workspace `settings.dlp.enabled = true`.

No AI tools — DLP rules are managed via REST only.

## Endpoints

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/ai/dlp/rules` | admin | List all DLP rules for the workspace |
| POST | `/ai/dlp/rules` | admin | Create a new DLP rule |
| PATCH | `/ai/dlp/rules/:id` | admin | Update an existing rule |
| DELETE | `/ai/dlp/rules/:id` | admin | Delete a rule |

### POST / PATCH body fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `rule_type` | string | yes (POST) | One of: `keyword`, `regex`, `type_block`, `llm_classify` |
| `value` | string | yes (POST) | Rule value — keyword, regex pattern, typeId, or LLM prompt |
| `description` | string | no | Human-readable description |
| `severity` | string | no | `block` (default), `warn`, `audit` |
| `enabled` | boolean | no | Default `true` |
| `model` | string | no | LLM alias for `llm_classify` rules (default: `fast`) |
| `timeout_ms` | integer | no | Timeout for `llm_classify` (default: 5000) |

## Rule Types

| Type | Value field | Behavior |
|------|-------------|----------|
| `keyword` | Plain text keyword | Case-insensitive substring match in outbound messages |
| `regex` | Regular expression | Regex match (text truncated to 100KB for safety) |
| `type_block` | EAV type ID | Blocks requests when agent context includes this type |
| `llm_classify` | System prompt | Sends text to an internal LLM for classification; fail-closed on errors |

## Severity Levels

| Severity | Behavior |
|----------|----------|
| `block` | Request is stopped, user receives explanation |
| `warn` | Request passes, warning is logged |
| `audit` | Request passes, match is logged silently |

## Engine (`contour-guard.js`)

- `checkDlp(workspaceId, body, contextTypes)` — main entry point
- Rules cached with 60s TTL, keyed by the **numeric** workspace id
- `invalidateCache(workspaceId)` — called after CRUD operations; accepts either the numeric id
  or the db name
- `llm_classify` rules use internal providers only (never external — avoids recursive DLP)
- Processing order: keywords, regex, type_block, then llm_classify (skipped if already blocked)
- Fail-closed: if `llm_classify` errors or times out, the rule triggers

### Two identifiers, one parameter

`_v2_dlp_rules.workspace_id` is an INTEGER referencing `_v2_workspaces.id`, but every caller of
`routedFetch` passes the workspace **db name** as `opts.workspaceId` (`ctx.db` in the agent
runner, `ev.db` in AI column listeners, `db` in file processing). The same value also feeds
`logLlmCall`, where `llm_calls.workspace_id` is TEXT — so token accounting worked with the
name while the rules query did not.

`resolveWorkspaceId()` bridges the two: a number (or a numeric string) is used as is, anything
else is looked up once in `_v2_workspaces` by `db_name` or `slug` and cached. Before that, a
DLP-enabled workspace made Postgres reject the rules query on type conversion, and since the
DLP block in `routedFetch` is not wrapped in `try`, the error propagated and killed the whole
LLM call — DLP did not filter requests, it broke them. The failure was dormant in production
only because no workspace had rules (`SELECT count(*) FROM _v2_dlp_rules` → 0 on 2026-08-12).

The rules cache is keyed by the resolved numeric id for the same reason: the CRUD routes call
`invalidateCache(req.workspace.id)`, so a cache keyed by db name would never have been
invalidated.

### What DLP can and cannot see

Rules match the text of the outgoing request. `extractText()` walks a multimodal message part
by part, taking `text` and skipping attachments (`image_url`, `image`, `input_audio`, `audio`,
`file`, `document`).

That means **the contents of an image or a document are not inspected**. A scan sent to a
vision model for OCR passes DLP with only its prompt examined. This is a deliberate limit, not
an oversight: before parts were split out, `JSON.stringify` put the whole base64 payload into
the matched text, where `keyword` rules fired on accidental matches inside the encoding and an
`llm_classify` rule shipped 10 000 characters of base64 to a model — and, being fail-closed,
blocked the request when that call timed out.

What protects documents in a restricted workspace is `settings.closed_contour`, which keeps
external providers out of the routing entirely. DLP rules are a filter on prompts, not on
attachments.

## DB Table

```sql
CREATE TABLE _v2_dlp_rules (
  id SERIAL PRIMARY KEY,
  workspace_id INTEGER NOT NULL REFERENCES _v2_workspaces(id) ON DELETE CASCADE,
  rule_type VARCHAR(32) NOT NULL,
  value TEXT NOT NULL,
  description TEXT,
  severity VARCHAR(16) NOT NULL DEFAULT 'block',
  enabled BOOLEAN NOT NULL DEFAULT true,
  model TEXT,
  timeout_ms INTEGER DEFAULT 5000,
  created_by INTEGER,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX _v2_dlp_rules_ws ON _v2_dlp_rules(workspace_id, enabled);
```

Schema defined in `src/shared/pg-schema.js`.
