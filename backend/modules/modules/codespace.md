# Module: codespace

**Path:** `src/api/v2/modules/codespace/`
**Files:** `router.js`, `service.js`, `git-server.js`, `git-hooks/`, `evidence-card.js`, `machine-gate.js`, `github-crypto.js`, `github-listeners.js`, `github-sync.js`, `server-functions.js`, `server-fn-executor.js`
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
| GET | `/codespace/:slug/commits` | List commits (`?ref=`, `?limit=`, `?offset=`, `?path=` — история одного файла, `git log -- path`) |
| GET | `/codespace/:slug/tags` | List tags `[{ name, sha, date }]`, свежие сверху; сбой git → `[]` |
| POST | `/codespace/:slug/tags` | Create lightweight tag (`name`, `ref`) — editor; дубль → 409, плохое имя/ref → 400 |
| DELETE | `/codespace/:slug/tags/:name` | Delete tag (editor; wildcard, поддерживает `release/v1`) |
| GET | `/codespace/:slug/tree/:ref/*` | Browse file tree at ref and path |
| GET | `/codespace/:slug/blob/:ref/*` | Get file content — text: `{ content, encoding }` JSON; binary: `application/octet-stream` |
| — | `:ref` может содержать слэш (`sandbox/1`) — фронт-сервис кодирует `encodeURIComponent(ref)`, роут декодирует; сырой слэш трактуется как начало path |
| GET | `/codespace/:slug/diff/:base/:head` | Diff between two refs |
| POST | `/codespace/:slug/commit` | Commit a single file (`branch`, `filePath`, `content` up to 50 MB, `message`) |
| POST | `/codespace/:slug/commit-multi` | Commit multiple files atomically (`branch`, `files: [{filePath, content}]` up to 100 files / 50 MB each, `message`) |
| POST | `/codespace/:slug/branches` | Create branch (`name`, `fromRef`) — editor |
| POST | `/codespace/:slug/branches/ff-push` | CAS fast-forward ветки (`branch`, `commit`, `expectedBase`) — editor; см. «ff-push» ниже |
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
| POST | `/codespace/:slug/pulls/:number/revert` | Revert merged PR (editor) — `git revert` merge-коммита (`-m 1` для merge, без `-m` для squash) в отцепленном worktree; синтакс-гейт как в mergePR; атомарная замена ссылки (CAS) — при конфликте/гонке 409, ветка не двигается; предок-гейт: если merge-коммит перестал быть в истории цели — 409. Возвращает `{ reverted, revertCommitSha }`, эмитит `codespace.pr.reverted` |
| GET | `/codespace/:slug/pulls/:number/comments` | List PR comments |
| POST | `/codespace/:slug/pulls/:number/comments` | Add PR comment (viewer+) |

Детектор не считает «стиранием» три случая: строка, чей blame-коммит старше
точки ветвления цели и исходной ветки (ветка знала строку до развилки и
заменила её сама, PM-952); строка, чей текст целиком вошёл в начало
добавленной строки (расширение, PM-1001 — такие попадают в
`details.extensions` отказа, но не блокируют); строка автора PR. Потеря строки
головного коммита цели (age=0) блокирует с формулировкой «зона конфликта».

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

**Protected branches.** `editor` is the floor, not the whole rule: a branch listed in `_v2_git_repos.protected_branches` (comma-separated) accepts writes **and deletions** only from `owner` or `admin`. Empty column — no branch is protected. Set via `PATCH /codespace/:slug/write-mode` (`admin` role, body `{ protectedBranches: string[] }` → `setWriteMode` in `service.js`).

- Git push path: the pre-receive hook posts the refs to `POST /api/v2/internal/git/hook/pre-receive`, which calls `isBranchPushAllowed(repo, ref, role)` (`git-server.js`) per ref and refuses the push naming the blocked branches. Only `refs/heads/*` are judged; tags pass through.
- REST/AI write paths: `assertBranchWritable` / `assertMergeAllowed` (`service.js`) raise 403 `BRANCH_PROTECTED`. A caller that supplies no actor role is let through and logged as `branch_protection_skipped` — the check is deliberately not fail-closed there.

**Auth:** HTTP Basic — username = anything, password = JWT token or service key.

**Клонирование/пуш (PM-163).** Точный URL:
`https://<хост>/api/v2/<workspaceSlug>/git/<repoSlug>.git` — канонический маршрут
(`gitServerMiddleware`, regex `^\/([^/]+)\/git\/([^/]+\.git)(\/.*)?$/` в `src/api/v2/index.js`).
Алиас `https://<хост>/api/v2/codespace/<workspaceSlug>/<repoSlug>.git`
(`mountCodespaceGitAlias` в `git-server.js`, mounted on the v2 router) ведёт туда же.
Вариант без префикса `/api/v2` (`https://<хост>/codespace/<workspaceSlug>/<repoSlug>.git`)
работает только на legacy дев-старте (`scripts/start.js` вешает алиас прямо на app);
на проде nginx такого location не имеет и отвечает SPA. Пароль в Basic-авторизации —
JWT-токен или служебный ключ (`smk_...`):

```
git clone https://<user>:<token>@<хост>/api/v2/<workspaceSlug>/git/<repoSlug>.git
```

**Auto-create:** pushing to a non-existent repo auto-creates it (editor+ only).

**Module guard:** returns 403 if `workspace.settings.modules.codespace === false`. Since TD-111 there is no platform default for `codespace` (it is opt-out): absence of the key means enabled, and a recorded `false` is always the owner's decision. The same guard (`requireModule('codespace')`) stands on the portal blob route and on the code-studio routes, so the portal door no longer serves blobs that the REST door refuses (ADR-027). Legacy platform-written `false` values were erased by a one-time migration at deploy (`registry/tables.js`, marker comment on `_v2_workspaces`).

**Repo creation guard:** `createRepo` refuses with 503 `NOT_CONFIGURED` when `GIT_ROOT` is not set on the server (TD-111) — this covers both the REST path and auto-create on first push.

**Concurrency:** max 20 parallel git processes via `p-limit`.

### Internal hook endpoint

`POST /api/v2/internal/git/hook/post-receive` — called by the post-receive git hook via `curl`. Requires `INTEGRAM_INTERNAL_TOKEN` Bearer header. Updates `size_bytes` in DB and emits `codespace.push` event.

## Git Hooks (`git-hooks/`)

- **pre-receive** — two checks, not one. (1) 500 MB size limit: incoming pack and combined (existing + incoming) size. (2) Branch protection: the hook posts the pushed refs and the actor to `POST /internal/git/hook/pre-receive` (`api/v2/index.js`, Bearer `INTERNAL_GIT_TOKEN`), which resolves the actor's role and refuses refs rejected by `isBranchPushAllowed`. **A failure of check (2) does not block the push**: the handler answers `allowed: true` on its own error on purpose — a broken check must not lock the repository, and the API write paths still enforce protection. Returns the error to the git client when a check refuses.
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
| `codespace.pr.reverted` | Merged PR reverted via `/revert` (payload: `revertCommitSha`) |
| `codespace.push` | Push received (post-receive hook) |
| `codespace.commit.created` | File committed via REST API (`commitFile`, `commitMultipleFiles`) |

## DB Tables (per-workspace, lazy-init)

**`_v2_git_repos`**
`id`, `name`, `slug`, `description`, `disk_path`, `default_branch`, `size_bytes`, `created_by`, `created_at`, `github_remote_url`, `github_token_enc` (AES-256-GCM encrypted PAT), `github_sync_dir` (push_only/pull_only/both), `github_auto_sync` (boolean), `github_webhook_secret` (HMAC secret for webhook verification), `github_last_sync` (timestamp of last sync), `github_last_error` (last sync error message), `protected_branches` (comma-separated branch names, default `''` — see Protected Branches), `require_base_commit` (boolean — when set, writing over an existing branch must carry `baseCommit`, else 409 `BASE_COMMIT_REQUIRED`), `required_gates` (comma-separated gate names `machine`/`review`, default `''` — see Required Gates)

All three write-mode columns are added lazily by `ensureWriteModeColumns()` (`service.js`), not by the initial `CREATE TABLE`.

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
| `ff_push(slug, branch, commit, expectedBase)` | HIGH | CAS fast-forward ветки на существующий коммит (см. «ff-push» ниже) |
| `get_file_tree(slug, ref, path)` | LOW | List directory tree |
| `read_blob(slug, ref, path, offset, limit)` | LOW | Read file content (supports pagination) |
| `get_blame(slug, ref, path)` | LOW | Get git blame annotations |
| `get_diff_range(slug, from, to)` | LOW | Diff between two refs |
| `get_evidence_card(slug)` | LOW | Get evidence card for repo |
| `create_repo(name, description)` | HIGH | Create new git repository |
| `delete_repo(slug)` | HIGH | Delete git repository |
| `remove_github_sync(slug)` | HIGH | Remove GitHub sync configuration |

MCP group: `codespace`. Activate in MCP via `search_tools("codespace")`.

**Merge conflict resolution (MERGE_CONFLICT):** the response is self-sufficient — `details.conflicts` (files), `details.blocks` (each block: your version / original between `|||||||` and `=======` / branch version), `details.baseCommit` (write the resolution against this state). Procedure: for each block write out both sides' intents vs the original (a deleted line is an intent — do not resurrect it), merge the two intents (mechanical concatenation is the typical mistake), commit via `commit_file` with `baseCommit`. `WRITE_CONTENTION` (`retryable=true`) needs no analysis — just retry.

### ff-push (PM-950)

`POST /codespace/:slug/branches/ff-push` и AI-тул `ff_push(slug, branch, commit, expectedBase)` (TIER_HIGH) — атомарная перемотка ветки на уже существующий коммит (`codespace/service.js` → `ffPushBranch`).

Контракт (по образцу GitHub PR update-branch `expected_head_sha` и GraphQL `updateRefs.beforeOid`):

- ветка сдвигается ТОЛЬКО если её текущая вершина равна `expectedBase` (точный CAS через `git update-ref refs/heads/<b> <new> <expected>`) и `commit` — её потомок (`merge-base --is-ancestor`);
- слияние не выполняется, история не переписывается, ни один коммит не теряется;
- возвращает `{ ok: true, slug, branch, old, new }`.

Отказы: `REF_STALE` 409 («ветка уехала», `details.currentBase` — свежий sha для повтора), `NOT_FAST_FORWARD` 409 (exit-код 1 от merge-base), `NOT_FOUND` 404 (нет ветки/коммита), `VALIDATION_ERROR` 400 (нет доводов / не-sha, `^[0-9a-f]{4,40}$`), `GIT_ANCESTRY_CHECK_FAILED` 500 (сбой git при проверке родства — только сбой, не «не потомок»).

Защищённые ветки: ff_push допущен к ним при точном CAS (Решение 3 PM-950, подтверждено владельцем) — в отличие от прямых записей (`assertBranchWritable`) и наравне с вливанием PR: `BRANCH_PROTECTED` не проверяется. Мотив — BRANCH_PROTECTED вынуждал агентов идти в обход через `create_pr`→`merge_pr` даже для тривиальной перемотки.

## Обязательные гейты (required gates)

Репозиторий может объявить гейты, без зелёного вердикта которых PR не мержится: `machine` (vitest + eslint на коде ветки) и `review` (AI-ревью диффа).

**Настройка:** `PATCH /codespace/:slug/write-mode` (`set_repo_write_mode` в MCP), body `{ requiredGates: ["machine", "review"] }` → колонка `_v2_git_repos.required_gates` (comma-separated, default `''` — гейтов нет). Пустой массив снимает все обязательные гейты; `undefined`/отсутствие поля — колонка не трогается. Допустимые имена: `machine`, `review`, иначе 400 `VALIDATION_ERROR`.

**Автозапуск:** при создании PR (`codespace.pr.created`) machine-гейт ставится в очередь `machine-gate` (BullMQ) автоматически — но только если `machine` входит в `required_gates` репозитория. Для репо без required gates ничего не запускается; ручной запуск — `POST /codespace/machine-gate`.

**Блокировка мержа:** `mergePR` перед слиянием проверяет каждый обязательный гейт по результатам в `_v2_pr_gate_results` (последний прогон данного `gate_type` для этого PR). Нарушение — 409 с кодом:
- `GATE_REQUIRED` — гейт для PR не прогонялся;
- `GATE_STALE` — гейт прогнан по другой вершине ветки-источника;
- `GATE_FAILED` — последний прогон красный.

**Обход:** `merge_pr` с `allowGateBypass: true` (зеркально `allowKnownLoss`) мержит мимо гейтов — явное действие звавшего, не молчаливое исключение.

**Привязка к sha:** результат гейта несёт `headSha` — sha вершины ветки-источника на момент прогона. Новый пуш инвалидирует прогон (см. `GATE_STALE`) — практика required checks: проверка привязана к коммиту, а не к PR.

**Исполнитель machine-гейта — host:** код репозитория исполняется на хосте (Docker на проде нет, решение владельца 14.10.2026). Защита: ephemeral detached worktree (`git worktree add --detach`, стирается в `finally`), белый список `GATE_ENV` (PATH/HOME/CI/NODE_ENV — секреты воркспейса и сервера в прогон не попадают), `execFileSync` без shell, таймаут 5 минут на шаг, concurrency 1. Посылка — trusted-code-only: в кодспейс пишут только участники воркспейса. Шаги выбираются по содержимому репо: `npm ci` при `package.json`, `vitest run` при `vitest.config.js`, `eslint . --max-warnings 0` при `eslint.config.js`/`.eslintrc.*`.

**Результаты:** таблица `_v2_pr_gate_results` (per-workspace): `pr_id`, `gate_type` (`machine`/`review`), `verdict` (`pass`/`fail`/`error`), `result` JSONB (`{ verdict, headSha, steps: [{ name, pass, output }] }`), `created_at`. Чтение — `GET /codespace/machine-gate/:prId` (последний machine-прогон) и `GET /codespace/evidence-card/:prId` (объединяет оба гейта).

## Machine Gate — исполнитель

Machine gate прогоняет код ветки-источника PR (vitest, eslint) на хосте в ephemeral worktree — см. «Обязательные гейты» выше про изоляцию и посылку. Прежний Docker-исполнитель убран: `sandbox-runner.js` (контейнер `node:22-slim`) удалён — на проде Docker нет, и былых предпосылок для контейнера тоже (решение владельца 14.10.2026).

**Files:**
- `machine-gate.js` — BullMQ worker (очередь `machine-gate`), worktree management, orchestration, `saveGateResult`/`getGateResult`/`checkGateVerdict`

## Server Functions

Server-side JavaScript functions stored in codespace repos under `api/*.js`, executed in isolated V8 sandboxes. Separate from workspace-tools — different isolates, limits, and OOM policy.

**Files:**
- `server-functions.js` — loader, 1-min TTL cache, capability parser, cache invalidation (`invalidateServerFunctionCache`) wired into the write pipeline: `writeToBranch`, `writeToNewBranch`, `mergePR` (covers REST commits, AI tools `commit_file`/`patch_file`/`commit_multi_files`/`delete_repo_file`, teamchat cells, portal studio) plus manual GitHub pull; the push path (`handlePostReceive`) and its webhook remain. A failed edit and an unchanged commit leave the cache untouched. Known remainder: the automations worker keeps its OWN cache in its own process until the 60 s TTL expires (PM ai2o #745).
- `server-fn-executor.js` — V8 sandbox: fresh isolate per request, 64 MB memory, 15s/5s timeout

**Wall-clock budget per function:** the author declares it in the function body with a
`// timeout: N` header comment (seconds — parsed by `parseTimeout` in `server-functions.js:87-105`,
clamped into [5s, 180s]; out-of-range values are clamped loudly in the log, junk falls back to
the profile default). The caller cannot raise this budget — `timeoutCapMs` in
`server-fn-executor.js:215-219` can only LOWER the effective timeout, never lift it. The
profile rate-limit budgets (tg/pm/ai counters) are per-call and do not depend on the wall clock.

**Endpoint:** `POST /:db/portal/api/fn/:repo/:name` (in portal router, not codespace router). Requires `portalAuth('admin', 'owner', 'editor')`. Rate limit: 60/min.

**Capabilities:** declared via `// capabilities: query, write, fetch` header comment. Default: `query`. Bridge functions are the same as workspace-tools (`createBridge` from `automations/isolated-runner.js`), but with stricter per-execution rate limits (10 query, 10 mutation, 5 fetch, 3 ai, 5 teamchat, 0 agent).

**`portalLink` capability (login links):** `createLoginLink({ clientObjectId, ttlDays })` mints a
personal link that logs a portal customer in without a code — for messages sent to the customer's
own channel (loyalty app, email, a QR on the packing slip). It returns `{ token, path, expiresAt,
ttlDays, clientObjectId }`; the absolute URL is built by the caller, because the workspace knows its
portal domain and the backend does not. The object must be a record of the portal's client table (`auth.clientsTypeId`), the TTL is clamped
to [1, 7] days (default 1), and the token is exchanged for a session at
`POST /:db/portal/api/auth/link/consume`. The link is **one-time**: the first consume burns it in
Valkey (`portal_link_used:{db}:{jti}`, the key lives until the link's exp), a repeated consume gets
401 `LINK_USED`. Redis unavailable — the link works repeatedly (fail-open). The session after login
is a portal_jwt for 30 days. Re-minting a link (`createLoginLink`) is the normal way to hand the
client a new login; the old link is not revoked by it — a specific link cannot be revoked by
deleting its burn key (it only appears after the first login), revoke the client's session or
re-mint the link instead.
**Short link for SMS:** `createLoginLink({ clientObjectId, ttlDays, short: true })` additionally
returns `code` and `shortPath` (`/portal/l/<code>`, 8 base62 chars): the token is stored in Valkey
under `portal_link_code:{db}:{code}` until the link's exp (`SET … EX … NX`). The full address with
the token is ~350 chars and takes 6 SMS by itself; the short one fits a single message next to a short
text. `GET /:db/portal/l/:code` (portal server route) only redirects to `/:db/portal/auth-link?c=<code>`,
and the page exchanges the code with the same consume (`{ code }`) — a GET from a link-preview robot
does not burn the link. It is the same link, not a second one: logging in by the code burns the
token's jti, and vice versa. Valkey unavailable at minting — no `shortPath` in the result (not an
error: the full link stays valid); at consume — 503 `SERVICE_UNAVAILABLE`.
Billed against the `mutation` counter; tier HIGH — this hands out access, not data.

**Key differences from workspace-tools:**

| Property | Workspace Tools | Server Functions |
|----------|----------------|------------------|
| Storage | `_v2_workspace_tools` DB table | `api/*.js` files in codespace repo |
| Isolate | Cached per workspace (Phase 1) | Fresh per request (always) |
| Memory | 128 MB | 64 MB |
| OOM | `process.exit(1)` | Graceful dispose |
| Timeout | 5s / 30s | 5s / 15s |
| Caller | AI agent / MCP | Portal custom code (`api.callFunction`) |

Full reference: [`docs/guides/codespace-server-functions.md`](../../../docs/guides/codespace-server-functions.md)

## Репозиторий, раздаваемый порталом

Если слаг репозитория указан в конфиге модуля `custom_code` любого портала
воркспейса, файлы этого репозитория читаются **по HTTP браузером посетителя**:
`GET /:db/portal/api/codespace/:repo/blob/:ref/*path`. Маршрут живёт в
портальном роутере, а не здесь, но раздаёт он именно codespace-репозиторий —
поэтому названо и тут.

Решают четыре заслона (`portal/code-guard.js` + обработчик в `portal/router.js`):
репозиторий должен быть назван конфигом (`REPO_NOT_IN_CONFIG`), ref — тоже
конфигом (`REF_NOT_ALLOWED`; произвольный sha из истории не читается), роль
клиента должна видеть сам модуль (`AUTH_REQUIRED` / `ROLE_REQUIRED`), формат —
входить в белый список `BLOB_ALLOWED_EXTENSIONS` (`FILE_TYPE_NOT_ALLOWED`).
Файлы `api/*.js` закрыты отдельным правилом: их исполняет сервер, компонентам
они не нужны.

Решение считается **на обращение**, а не на репозиторий целиком: круг решающих
модулей сужается по запрошенной ветке всегда, а по имени файла — когда оно
названо входом какого-то модуля этой ветки (относительные импорты компонента
конфигом не названы и решаются всей веткой). Иначе публичный модуль открывал бы
анониму файлы ролевого модуля того же репозитория — так и было до 22.08.2026,
воспроизведено ответом 200 в `portal/__tests__/blob-route-access.test.js`.
Довод и разбор случая «файл назван двумя модулями» — в `portal/code-guard.js`.

**Что этими заслонами НЕ закрыто, и это решение, а не упущение:**

- `POST /:db/portal/api/fn/:repo/:name` — исполнение серверной функции. Закрыт
  ролью портала (`portalAuth('admin','owner','editor')`) и лимитом 60/мин, но не
  выключателем модуля `codespace`. Умолчание выключателя — `false`
  (`workspaces/service.js`), то есть по нему нельзя отличить «владелец выключил»
  от «никто не включал»; постановка выключателя погасила бы работающие порталы.
  Замер 22.08.2026: вызовов из компонентов боевых репозиториев — 142
  (`git grep -E "callFunction|callFn|/api/fn/" HEAD` по каждому bare-репозиторию;
  из них 27 — в клоне-стенде). То же и для чтения кода студией.
- Белого списка форматов у студии кода (`GET /api/code/tree`, `/api/code/file`)
  нет намеренно: её пользователь — сотрудник, участник воркспейса, и REST-дверь
  кодспейса отдаёт ему те же файлы без списка. Список сделал бы портальную дверь
  строже REST; живые обращения читают `README.md` и `api/*.js` — ровно то, что
  белый список закрывает.

Удаление файла коммитом доступ не закрывает, пока ref тот же: содержимое
читается из истории, пока объект достижим. Чистка — `git filter-repo`, затем
`git reflog expire --expire=now --all && git gc --prune=now` в bare-репозитории.

Что раздаётся сейчас: `cd backend && node scripts/codespace-blob-exposure.mjs`.
Обход идёт по всем портальным конфигам, включая воркспейсы с выключенным
`codespace` (отбор по выключателю уже приводил к тому, что боевые репозитории
молча выпадали из замера), различает «закрыто заслоном» и «файла нет», следует
за относительными импортами компонента и отдельно проверяет страницу
сотрудников, зовущую маршрут мимо конфига. Код возврата: `0` — заслон живого не
задевает, `1` — задевает (разбирать до выкладки), `2` — измерить не удалось.

## Recent fixes

- **Revert merged PR / история файла / теги** (22.10.2026, ветка codespace-fixes) — `POST /:slug/pulls/:number/revert`, `?path=` в `/commits`, `/tags` CRUD. UI: кнопка «Откатить PR» в дровере merged-PR, фильтр «История» у файла на вкладке коммитов, блок «Теги» в AboutSidebar. Список веток начинается с `default_branch` (клиент берёт branches[0]); карточка списка репо показывает `last_commit_at` (последний коммит дефолтной ветки) вместо даты создания. Тесты: `__tests__/git-completeness.test.js` (15 кейсов на настоящем git).
- **PR comment authors** — `listComments` now JOINs `_v2_users` and returns `author_username`, `author_email` alongside `author_id`.
- **Initial commit diff** — `GET /:slug/commits/:sha/diff` uses `git show --format=` and works for all commits including the initial one.
- **Merge strategy** — `POST /:slug/pulls/:number/merge` body accepts `strategy` (merge/squash/rebase) to override the PR default.
- **Merge commit diff** — `getCommitDiff` detects merge commits (2 parents via `git rev-parse --verify sha^2`) and uses `git diff sha^1 sha^2` instead of `git show`, which returns an empty combined diff for clean merges. The PR drawer uses this endpoint for merged PRs (`merge_commit_sha`) to show the actual introduced diff.
- **Open PR diff argument order** — `PrDrawer` calls `getDiff(target_branch, source_branch)` (base=target, head=source) so the three-dot diff is `git diff target...source` — shows only feature branch changes. Previously the args were swapped, returning an empty diff for open PRs.
- **Git Smart HTTP** — `gitServerMiddleware` (from `git-server.js`) is now mounted in `src/api/v2/index.js` before the `/:db` JWT middleware, enabling `git clone`/`push` via HTTP Basic Auth (password = JWT token).
- **Multi-file atomic commit** — `POST /:slug/commit-multi` and `commitMultipleFiles` service function commit up to 100 files in a single git commit via worktree. Each file is path-traversal-checked. Tool `commit_multi_files` available in AI agent (TIER_MEDIUM).
- **commitFileTool / deleteRepoFileTool commitHash fix** — these tools previously returned `result.hash || result.commitHash` (always null) instead of `result.sha`. Fixed to return `result.sha`.
