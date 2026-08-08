# Module: codespace

**Path:** `src/api/v2/modules/codespace/`
**Files:** `router.js`, `service.js`, `git-server.js`, `git-hooks/`, `evidence-card.js`, `machine-gate.js`, `sandbox-runner.js`, `github-crypto.js`, `github-listeners.js`, `github-sync.js`, `server-functions.js`, `server-fn-executor.js`
**Base URL:** `/api/v2/:db/codespace/...`
**Auth:** JWT required. Write: `editor`. Delete repo: `admin`. PR comments: `viewer`. Git HTTP: HTTP Basic (password = JWT or service key).

## Purpose

Git repository hosting per workspace. Each workspace can have one or more Git repos stored on the filesystem. Supports standard Git Smart HTTP protocol (clone, push, pull), pull requests, branch management, and hooks. 500 MB size limit per repo.

## Endpoints

### Repository management

Repos are identified by `slug` (not numeric ID).

| Method | Path | Description |
|--------|------|-------------|
| GET | `/codespace` | List repositories |
| POST | `/codespace` | Create repository (`name`, `slug`, `description`) |
| GET | `/codespace/:slug` | Get repo info (default branch, size, last commit) |
| DELETE | `/codespace/:slug` | Delete repository (admin) |

### Git content

| Method | Path | Description |
|--------|------|-------------|
| GET | `/codespace/:slug/branches` | List branches |
| GET | `/codespace/:slug/commits` | List commits (`?ref=`, `?limit=`, `?offset=`) |
| GET | `/codespace/:slug/tree/:ref/*` | Browse file tree at ref and path |
| GET | `/codespace/:slug/blob/:ref/*` | Get file content — text: `{ content, encoding }` JSON; binary: `application/octet-stream` |
| GET | `/codespace/:slug/diff/:base/:head` | Diff between two refs |
| POST | `/codespace/:slug/commit` | Commit a single file (`branch`, `filePath`, `content` up to 50 MB, `message`) |
| POST | `/codespace/:slug/commit-multi` | Commit multiple files atomically (`branch`, `files: [{filePath, content}]` up to 100 files / 50 MB each, `message`) |
| POST | `/codespace/:slug/branches` | Create branch (`name`, `fromRef`) — editor |
| DELETE | `/codespace/:slug/branches/:name` | Delete branch (editor; wildcard, supports `feature/foo`) |
| DELETE | `/codespace/:slug/blob` | Delete file (`branch`, `filePath`, `message` optional — default: "Delete {filePath}") — editor |
| GET | `/codespace/:slug/commits/:sha/diff` | Diff for single commit. For merge commits (2 parents) uses `git diff sha^1 sha^2`; otherwise `git show --format=` |
| GET | `/codespace/:slug/blame/:ref/*` | Get git blame for a file |
| POST | `/codespace/:slug/tree-commits` | Deferred last-commit-per-file for tree entries |

### Pull requests

Pull requests are identified by sequential number within a repo.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/codespace/:slug/pulls` | List pull requests (`?status=open\|closed\|merged\|draft`, `?limit=`, `?offset=`) |
| POST | `/codespace/:slug/pulls` | Create PR (`title`, `sourceBranch`, `targetBranch`, `mergeStrategy`) |
| GET | `/codespace/:slug/pulls/:number` | Get PR details |
| PATCH | `/codespace/:slug/pulls/:number` | Update PR (`title`, `description`, `status`, `mergeStrategy`) — cannot update merged PR |
| POST | `/codespace/:slug/pulls/:number/merge` | Merge PR (checks conflicts first; supports merge/squash/rebase strategies); body: `{ strategy? }` — overrides PR merge strategy |
| GET | `/codespace/:slug/pulls/:number/comments` | List PR comments |
| POST | `/codespace/:slug/pulls/:number/comments` | Add PR comment (viewer+) |

### Gate and Evidence

| Method | Path | Role | Description |
|--------|------|------|-------------|
| POST | `/codespace/review-gate` | editor | Adversarial multi-agent review of PR diff (`repo`, `pr`). Runs reviewer + sandbox-opponent via council, returns structured verdict. |
| POST | `/codespace/machine-gate` | editor | Trigger machine gate (lint, tests) for a PR |
| GET | `/codespace/machine-gate/:prId` | viewer | Get machine gate results for a PR |
| GET | `/codespace/evidence-card/:prId` | viewer | Get aggregated evidence card for a PR |

### GitHub Sync

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/codespace/:slug/github` | viewer | Get GitHub sync config (token masked) |
| PATCH | `/codespace/:slug/github` | editor | Configure GitHub sync (remoteUrl, token, direction, autoSync) |
| DELETE | `/codespace/:slug/github` | editor | Remove GitHub sync config |
| POST | `/codespace/:slug/github/push` | editor | Manual push to GitHub |
| POST | `/codespace/:slug/github/pull` | editor | Manual pull from GitHub |
| POST | `/codespace/:slug/github/webhook` | — | GitHub push webhook (no JWT, HMAC-SHA256 verified) |

**GitHub Sync Details**

Optional per-repo mirror to a GitHub repository. PAT stored AES-256-GCM encrypted (key derived from `JWT_SECRET` via scrypt). Token is never written to disk — always passed inline in the git HTTPS URL.

**Modes:**
- **push_only** — every local commit triggers `git push --force` to GitHub. Sources: git HTTP push (via `codespace.push` event) and REST commits (`/commit`, `/commit-multi`). Webhook not used.
- **pull_only** — GitHub pushes trigger our webhook, which runs `git fetch +refs/heads/*:refs/heads/*` (force-update all branches). Webhook secret verified via HMAC-SHA256. Auto-pull on webhook or manual via `/github/pull` endpoint.
- **both** — bidirectional sync. Local commits auto-push to GitHub (`push_only` behavior) **and** GitHub pushes trigger auto-pull via webhook (`pull_only` behavior). Both Push and Pull manual buttons are shown in the UI.

**Push implementation note:** Uses `--force` (not `--force-with-lease`). Bare repos without a named remote have no tracking refs, so `--force-with-lease` always fails with "stale info" on the second push.

**Files:**
- `github-crypto.js` — `encryptToken` / `decryptToken` — AES-256-GCM encryption/decryption with scrypt key derivation
- `github-sync.js` — `configureSync`, `getGithubConfig`, `removeGithubSync`, `pushToGithub`, `pullFromGithub` — main sync operations
- `github-listeners.js` — `registerListeners(bus)` — auto-push on `codespace.push` event when direction is `push_only` or `both`

**Webhook setup (pull_only or both mode):**
1. Configure pull_only or both in Integram UI → Webhook URL and Secret are shown
2. GitHub repo → Settings → Webhooks → Add webhook
3. Payload URL: `<your-integram-url>/api/v2/<workspace>/codespace/<slug>/github/webhook`
4. Content type: `application/json`
5. Secret: the value shown in Integram UI
6. Events: Just the `push` event

**Webhook route location:** The webhook endpoint is registered in `src/api/v2/index.js` **before** `requireJwt`, making it public (no JWT needed). GitHub signature (HMAC-SHA256 `X-Hub-Signature-256` header) is verified before any action is taken.

### Git Smart HTTP (`git-server.js`)

**URL:** `/api/v2/:workspaceSlug/git/:repoSlug.git[/*]` — mounted in `src/api/v2/index.js` before `/:db` middleware (has its own Basic Auth, bypasses JWT middleware).

Implements the Git Smart HTTP protocol via `git http-backend` subprocess.
- `git-upload-pack` — fetch/clone
- `git-receive-pack` — push (requires `editor` role)

**Auth:** HTTP Basic — username = anything, password = JWT token or service key.

**Auto-create:** pushing to a non-existent repo auto-creates it (editor+ only).

**Module guard:** returns 403 if `workspace.settings.modules.codespace === false`.

**Concurrency:** max 20 parallel git processes via `p-limit`.

### Internal hook endpoint

`POST /api/v2/internal/git/hook/post-receive` — called by the post-receive git hook via `curl`. Requires `INTEGRAM_INTERNAL_TOKEN` Bearer header. Updates `size_bytes` in DB and emits `codespace.push` event.

## Git Hooks (`git-hooks/`)

- **pre-receive** — enforces 500 MB size limit. Checks both incoming pack and combined (existing + incoming) size. Returns error to git client if exceeded.
- **post-receive** — calls internal hook endpoint to update size and emit push event.

## Storage

Repos stored on filesystem. Env var: `GIT_ROOT` (default `/data/git`).
Disk path format: `{GIT_ROOT}/{workspaceDb}/{repoId}.git` — ID-based, not slug-based.

On repo creation: bare git init + initial commit with `README.md` is created automatically.

## Size Limit

500 MB per repo, enforced on push via pre-receive hook.

## Events

| Event | When |
|---|---|
| `codespace.pr.created` | PR created |
| `codespace.pr.merged` | PR merged |
| `codespace.push` | Push received (post-receive hook) |
| `codespace.commit.created` | File committed via REST API (`commitFile`, `commitMultipleFiles`) |

## DB Tables (per-workspace, lazy-init)

**`_v2_git_repos`**
`id`, `name`, `slug`, `description`, `disk_path`, `default_branch`, `size_bytes`, `created_by`, `created_at`, `github_remote_url`, `github_token_enc` (AES-256-GCM encrypted PAT), `github_sync_dir` (push_only/pull_only/both), `github_auto_sync` (boolean), `github_webhook_secret` (HMAC secret for webhook verification), `github_last_sync` (timestamp of last sync), `github_last_error` (last sync error message)

**`_v2_git_pull_requests`**
`id`, `repo_id`, `number`, `title`, `description`, `status`, `source_branch`, `target_branch`, `merge_strategy`, `merge_commit_sha`, `author_id`, `created_at`, `merged_at`

**`_v2_git_pr_counter`**
`repo_id`, `next_number` — atomic sequential PR numbering

**`_v2_git_pr_comments`**
`id`, `pr_id`, `author_id`, `body`, `created_at`

## AI Agent Tools

All codespace operations are available as AI agent tools (both in-app agent and MCP). Tool file: `modules/ai/agent/tools/codespace.js`.

| Tool | Tier | Description |
|---|---|---|
| `list_repos` | LOW | List repos in workspace |
| `get_repo(slug)` | LOW | Repo info |
| `list_branches(slug)` | LOW | List branches |
| `list_commits(slug, ref?)` | LOW | List commits |
| `get_commit_diff(slug, sha)` | LOW | Diff for one commit |
| `list_prs(slug, status?)` | LOW | List pull requests |
| `get_pr(slug, number)` | LOW | PR details |
| `list_pr_comments(slug, number)` | LOW | PR comments |
| `create_branch(slug, name, fromRef?)` | MEDIUM | Create branch |
| `commit_file(slug, branch, filePath, content, message?)` | MEDIUM | Create/update single file |
| `commit_multi_files(slug, branch, files, message)` | MEDIUM | Commit multiple files atomically; `files` = `[{filePath, content}]` |
| `create_pr(slug, title, sourceBranch, targetBranch, description?, mergeStrategy?)` | MEDIUM | Create PR |
| `update_pr(slug, number, ...)` | MEDIUM | Update PR (title/description/status/strategy); `status="open"` reopens |
| `add_pr_comment(slug, number, body)` | MEDIUM | Add PR comment |
| `get_github_sync(slug)` | LOW | Get GitHub Sync config for repo |
| `configure_github_sync(slug, remoteUrl, token, direction, autoSync?)` | HIGH | Configure GitHub Sync (push_only/pull_only/both) |
| `push_to_github(slug)` | MEDIUM | Manually push all branches to GitHub |
| `pull_from_github(slug)` | MEDIUM | Manually pull all branches from GitHub |
| `delete_branch(slug, name)` | HIGH | Delete branch |
| `delete_repo_file(slug, branch, filePath)` | HIGH | Delete file via git rm |
| `merge_pr(slug, number, strategy?)` | HIGH | Merge PR |
| `get_file_tree(slug, ref, path)` | LOW | List directory tree |
| `read_blob(slug, ref, path, offset, limit)` | LOW | Read file content (supports pagination) |
| `get_blame(slug, ref, path)` | LOW | Get git blame annotations |
| `get_diff_range(slug, from, to)` | LOW | Diff between two refs |
| `get_evidence_card(slug)` | LOW | Get evidence card for repo |
| `create_repo(name, description)` | HIGH | Create new git repository |
| `delete_repo(slug)` | HIGH | Delete git repository |
| `remove_github_sync(slug)` | HIGH | Remove GitHub sync configuration |

MCP group: `codespace`. Activate in MCP via `search_tools("codespace")`.

## Machine Gate — Docker Sandbox

Machine gate runs user repository code (vitest, eslint) in a disposable Docker container for process isolation. No code executes on the host.

**Architecture:** `git worktree add` → Docker container → result → worktree cleanup

**Files:**
- `machine-gate.js` — BullMQ worker, worktree management, orchestration
- `sandbox-runner.js` — Docker container lifecycle (create → start → wait → logs → remove)

**Container config:**
- Image: `node:22-slim`
- Volume: worktree path bind-mounted as `/code:rw`
- Network: bridge (needed for `npm ci`)
- Env: `CI=1`, `NODE_ENV=test`, `HOME=/tmp` — no host secrets
- User: `1000:1000` (not root)
- Memory: 1 GB
- CPU: 2 cores
- PIDs: 256 max
- Timeout: 5 minutes
- `CapDrop: ALL`, `no-new-privileges: true`

**Prerequisites:**
- Docker installed: `apt install -y docker.io && systemctl enable docker`
- Image pulled: `docker pull node:22-slim`
- iptables rule to block container access to host services: `iptables -I DOCKER-USER -s 172.17.0.0/16 -d 172.17.0.1 -j DROP`

## Server Functions

Server-side JavaScript functions stored in codespace repos under `api/*.js`, executed in isolated V8 sandboxes. Separate from workspace-tools — different isolates, limits, and OOM policy.

**Files:**
- `server-functions.js` — loader, 1-min TTL cache, capability parser, cache invalidation on `codespace.push`
- `server-fn-executor.js` — V8 sandbox: fresh isolate per request, 64 MB memory, 15s/5s timeout

**Endpoint:** `POST /:db/portal/api/fn/:repo/:name` (in portal router, not codespace router). Requires `portalAuth('admin', 'owner', 'editor')`. Rate limit: 60/min.

**Capabilities:** declared via `// capabilities: query, write, fetch` header comment. Default: `query`. Bridge functions are the same as workspace-tools (`createBridge` from `automations/isolated-runner.js`), but with stricter per-execution rate limits (10 query, 10 mutation, 5 fetch, 3 ai, 5 teamchat, 0 agent).

**Key differences from workspace-tools:**

| Property | Workspace Tools | Server Functions |
|----------|----------------|------------------|
| Storage | `_v2_workspace_tools` DB table | `api/*.js` files in codespace repo |
| Isolate | Cached per workspace (Phase 1) | Fresh per request (always) |
| Memory | 128 MB | 64 MB |
| OOM | `process.exit(1)` | Graceful dispose |
| Timeout | 5s / 30s | 5s / 15s |
| Caller | AI agent / MCP | Portal custom code (`api.callFunction`) |

Full reference: [`docs/codespace-server-functions.md`](../../../docs/codespace-server-functions.md)

## Recent fixes

- **PR comment authors** — `listComments` now JOINs `_v2_users` and returns `author_username`, `author_email` alongside `author_id`.
- **Initial commit diff** — `GET /:slug/commits/:sha/diff` uses `git show --format=` and works for all commits including the initial one.
- **Merge strategy** — `POST /:slug/pulls/:number/merge` body accepts `strategy` (merge/squash/rebase) to override the PR default.
- **Merge commit diff** — `getCommitDiff` detects merge commits (2 parents via `git rev-parse --verify sha^2`) and uses `git diff sha^1 sha^2` instead of `git show`, which returns an empty combined diff for clean merges. The PR drawer uses this endpoint for merged PRs (`merge_commit_sha`) to show the actual introduced diff.
- **Open PR diff argument order** — `PrDrawer` calls `getDiff(target_branch, source_branch)` (base=target, head=source) so the three-dot diff is `git diff target...source` — shows only feature branch changes. Previously the args were swapped, returning an empty diff for open PRs.
- **Git Smart HTTP** — `gitServerMiddleware` (from `git-server.js`) is now mounted in `src/api/v2/index.js` before the `/:db` JWT middleware, enabling `git clone`/`push` via HTTP Basic Auth (password = JWT token).
- **Multi-file atomic commit** — `POST /:slug/commit-multi` and `commitMultipleFiles` service function commit up to 100 files in a single git commit via worktree. Each file is path-traversal-checked. Tool `commit_multi_files` available in AI agent (TIER_MEDIUM).
- **commitFileTool / deleteRepoFileTool commitHash fix** — these tools previously returned `result.hash || result.commitHash` (always null) instead of `result.sha`. Fixed to return `result.sha`.
