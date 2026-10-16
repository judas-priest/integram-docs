# Claude Code Skills & Agents

This document describes all custom skills and agents available in the project. They are shared via git — every team member gets them on `git pull`.

## Skills (`.claude/skills/*/SKILL.md`)

Skills are invoked via `/skill-name` in Claude Code. Some activate automatically when Claude detects a matching task context.

### Git Workflow

| Skill | Command | Auto-invoke | Description |
|-------|---------|-------------|-------------|
| **commit** | `/commit` | No (manual only) | Stage and commit changes following project git rules. Injects `git status` and `git diff` as context before processing. |
| **review** | `/review [path]` | Yes | Review uncommitted changes against all project rules (backend, frontend, portal, AI tools, git). Accepts optional path to focus scope. |
| **pr** | `/pr` | No (manual only) | Create a pull request. Checks GitHub username for push permissions. Injects branch info and diff stats. |
| **hotfix** | `/hotfix branch-name` | No (manual only) | Full hotfix workflow: create git worktree, fix, test, commit, push, create PR — all isolated from main working directory. |

### Code Audits

All audit skills are manual-only (`disable-model-invocation: true`) and accept a `days-back` argument to scope the audit window.

| Skill | Command | Description |
|-------|---------|-------------|
| **docs-audit** | `/docs-audit [days]` | Verify documentation matches code across all modules. Dispatches parallel subagents per module, then re-verifies findings. |
| **security-audit** | `/security-audit [days]` | OWASP Top 10 check: SQL injection, command injection, XSS, auth/access control. Parallel agents by category. |
| **dead-code** | `/dead-code` | Find unused exports, unreachable tool-executor cases, orphaned components. Reports with confidence levels. |
| **test-coverage** | `/test-coverage [days]` | Find untested service functions, tool-executor cases, router endpoints. Prioritized by criticality. |
| **dependency-check** | `/dependency-check` | Run `npm audit` and `npm-check-updates` across backend, frontend, portal. Flag critical vulnerabilities. |

### Development

| Skill | Command | Auto-invoke | Description |
|-------|---------|-------------|-------------|
| **add-module** | `/add-module module-name` | Yes | 6-step checklist for adding a new backend module: create files, register router, create docs, update OpenAPI. |
| **add-tool** | `/add-tool tool-name` | Yes | 6-step checklist for adding a new AI tool: handler, TOOL_DEFS, executor case, risk tier, agent prompt, MCP prompt. Includes workspace-tool alternative. |
| **sync-check** | `/sync-check` | Yes | Verify docs are in sync with code changes before committing. Maps changed files to their expected documentation. |

## Agents (`.claude/agents/*.md`)

Agents are specialized subagents that run in isolated context windows. Claude delegates to them automatically based on the task description, or you can request them explicitly (e.g., "use the backend-reviewer agent").

All agents use `claude-sonnet-4-6` (cost-efficient) and are restricted to read-only tools (`Read, Grep, Glob`).

| Agent | When it activates | What it checks |
|-------|-------------------|----------------|
| **backend-reviewer** | Reviewing backend changes, PRs with backend code | SQL safety (parameterized queries), module structure (router vs service), EAV correctness (up=1 vs up=parentId), error handling (AppError), ref patterns, middleware chain |
| **frontend-reviewer** | Reviewing frontend changes, Vue/JS PRs | Layer separation (services for HTTP, stores for state), PrimeVue explicit imports, Select @change gotcha, error handling (toast), v-for array safety, multiselect EAV pattern |
| **portal-reviewer** | Reviewing portal or telegram code | Portal: portal_jwt auth, config ID usage, credentials: include, CSP rules, custom_code bindings. Telegram: JSONB parsing, sendMessage signature, editMessageText gotcha, env-specific IDs, two renderers |
| **tool-auditor** | After adding/modifying AI tools | Sync verification across 4 files: TOOL_DEFS, tool-executor cases, risk-tiers entries, MCP system prompt references. Outputs a pass/fail table per tool. |

### Agent output format

All review agents return:
```
- VERDICT: pass | needs-fixes
- Findings: bulleted list, file:line specific
- Suggested fixes: one line per finding
```

Agents do not fix anything — they only report. The main session decides what to apply.

## Hooks (`.claude/hooks/`)

Hooks run automatically on Claude Code events. They enforce rules without manual invocation.

| Hook | Trigger | What it does |
|------|---------|-------------|
| **check-docs-sync** | PostToolUse (file edit) | Reminds to update docs when key files are edited (router.js -> module docs, TOOL_DEFS -> executor, etc.) |
| **auto-format** | PostToolUse (file edit) | Auto-formats backend .js files with prettier after edits |
| **check-tests-status** | Stop | Reminds to run tests if backend/frontend source files changed during the session |
| **block-sensitive-files** | PreToolUse (Write/Edit) | Blocks edits to .env, .git/, mcp*.json, ~/.claude/ settings |
| **guard-architecture** | PreToolUse (Write) | Blocks file creation in wrong locations (e.g., business logic in utils/, axios in components/) |

## Rules (`.claude/rules/`)

Rules are loaded automatically when editing files matching their `paths` pattern. They provide context-specific guidance.

| Rule | Paths | Key content |
|------|-------|-------------|
| **backend.md** | `backend/**` | Module structure, EAV queries, AppError, events, middleware order, gotchas |
| **frontend.md** | `frontend/**` | Vue 3 layers, services pattern, stores pattern, PrimeVue, error handling |
| **portal.md** | `portal/**` | Portal JWT auth, config IDs, EAV access, telegram bot gotchas |
| **ai-tools.md** | `backend/*/ai/**`, `mcp-server/**` | TOOL_DEFS as single source of truth, 7-step checklist, risk tiers, MCP+REST dual paths |
| **architecture.md** | `backend/**`, `frontend/**`, `portal/**` | Mandatory rules: read first, find right place, never invent patterns, module list |
| **git.md** | (always loaded) | Push permissions, commit rules, worktree usage, config files to never touch |
| **discussion-and-honesty.md** | (always loaded) | Discussion vs task distinction, honesty, anti-sycophancy |

### MCP-Powered Audits (integram)

Complex audit skills that use the integram MCP server to inspect live workspace data. All are manual-only and accept a workspace slug argument.

| Skill | Command | What it audits |
|-------|---------|---------------|
| **audit-schema** | `/audit-schema slug` | Orphaned refs, empty lookups, broken computed columns, duplicate aliases, child table consistency |
| **audit-automations** | `/audit-automations slug` | Broken triggers/actions, failing webhooks, stale schedules, telegram bot health |
| **audit-portal** | `/audit-portal slug` | Config bindings vs real schema, catalog data quality, order pipeline, telegram bot integration |
| **audit-permissions** | `/audit-permissions slug` | Over-permissioned users, missing grants, redundant roles, stale accounts |
| **audit-data** | `/audit-data slug` | Broken ref values, empty required fields, duplicate records, orphaned children, data freshness |

Each audit skill:
1. Switches to the target workspace via MCP
2. Collects schema/data/config through integram MCP tools
3. Dispatches 4-5 parallel subagents (one per check category)
4. Re-verifies agent findings via direct MCP calls (agents hallucinate)
5. Reports grouped by severity with actionable fixes

## Adding new skills/agents

### New skill

```bash
mkdir .claude/skills/my-skill
cat > .claude/skills/my-skill/SKILL.md << 'EOF'
---
name: my-skill
description: What it does and when to use it. Be specific — Claude uses this to auto-invoke.
allowed-tools: Read, Grep, Glob
argument-hint: [arg-name]
---

# My Skill

Instructions here. Use $ARGUMENTS for the argument.
EOF
```

Key frontmatter fields:
- `description` — **most important**. Claude reads this to decide when to auto-load the skill.
- `disable-model-invocation: true` — for destructive skills (commit, deploy, delete)
- `allowed-tools` — pre-approve tools so Claude doesn't ask permission each time
- `argument-hint` — shown in autocomplete when typing `/my-skill`

### New agent

```bash
cat > .claude/agents/my-agent.md << 'EOF'
---
name: my-agent
description: When to use this agent. Be specific.
tools: Read, Grep, Glob
model: claude-sonnet-4-6
---

You are a specialist in X. Your job is to...

## What you check
1. ...

## Output format
Return a markdown block with VERDICT and Findings.
EOF
```

Key rules:
- Use `tools` (not `allowed-tools`) — this is the agent format
- Restrict tools to minimum needed (never give Bash to a review agent)
- Use `claude-sonnet-4-6` for cost efficiency, `claude-opus-4-6` only for complex reasoning
- Keep agents narrow and specific — "backend SQL reviewer" beats "senior engineer"
