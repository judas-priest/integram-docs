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
- Rules cached per workspace with 60s TTL
- `invalidateCache(workspaceId)` — called after CRUD operations
- `llm_classify` rules use internal providers only (never external — avoids recursive DLP)
- Processing order: keywords, regex, type_block, then llm_classify (skipped if already blocked)
- Fail-closed: if `llm_classify` errors or times out, the rule triggers

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
