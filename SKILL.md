---
name: kanban-orchestrator
description: "Strict Kanban-first orchestrator for delegated builds, tasks, and feature workflows using agent-kanban-board. Use when managing tasks on a kanban board, orchestrating builder, reviewer, and tester agent workflows, or deploying the local agent-kanban-board server."
version: 2.14.7
---

# Kanban Orchestrator Protocol

Load this skill when a delegated build/feature task is initiated using an agent Kanban board. You are a **Strict Orchestrator (Admin)**. You do not write code directly; you manage the state machine, the board, and the workers.

> **Install this skill:** copy this `kanban-orchestrator/` directory into your opencode skill path — either project-local `.opencode/skills/kanban-orchestrator/` or global `~/.config/opencode/skills/kanban-orchestrator/`. The directory name must match the `name:` in the frontmatter above.

## 1. Environment & Local Deployment

Mandatory configuration fields:
- `kanban_url`
- `kanban_token`
- `project_name`

Check for these fields in the following order:
1. `.opencode/config.json` (Expected: `kanban_url`, `kanban_token`, `project_name`, `admin_token`)
2. Environment Variables: `KANBAN_URL`, `KANBAN_TOKEN`, `KANBAN_PROJECT`, `KANBAN_ADMIN_TOKEN`

### Automatic Local Deployment
If `kanban_url` is not provided in `.opencode/config.json` or environment variables, deploy the Kanban server locally:
1. Clone the board repository:
   ```bash
   git clone https://github.com/riazrahaman/agent-kanban-board.git
   cd agent-kanban-board
   ```
2. Install dependencies (the server and client each have their own manifest — there is no root `npm install` step):
   ```bash
   npm --prefix server install
   npm --prefix client install
   ```
3. Build the client and start the board server (the API serves the built SPA on the same port):
   ```bash
   npm run build
   npm start
   ```
   For a live-reloading client during development, run `npm --prefix client run dev` (Vite on `http://localhost:5173`) in a second terminal instead of `npm run build`.
4. Verify the local server port (default `http://localhost:4000` or indicated in stdout) and set `kanban_url`.
5. Obtain or configure the initial auth token to set `kanban_token`.

**CRITICAL**: If `project_name` or `kanban_token` cannot be determined after setup, you **MUST** halt and prompt the user to specify them.

**Verification Step**: Once mandatory fields are acquired, perform `GET /projects` using the `kanban_token` in headers to verify connectivity and project existence.

**Security**: Never commit configuration files. `.opencode/` holds authentication tokens. Confirm it is added to `.gitignore` before writing, and never stage `.opencode/` in git commits.

## 2. Project Scoping — `?project=` is MANDATORY on task paths

Unqualified paths silently resolve against the **default** project, which is almost never the target workspace. The failure is misleading: writes return `403 Forbidden: token is not authorized for project 'default'`, and reads return `404`.

Verified endpoint scoping behavior:

| Endpoint Call | Scoping Requirement | Notes |
|---|---|---|
| `POST /tasks` | In JSON Body: `{"project": "<project_name>", ...}` | Task creation is the only route taking project in body |
| `PATCH /tasks/:id?project=X` | Query parameter: `?project=X` | Required for status moves and updates |
| `PATCH /tasks/:id` (no `?project=`) | None | Fails with `403 Forbidden` |
| `POST /tasks/:id/claim?project=X` | Query parameter: `?project=X` | Required to claim ownership and lease |
| `POST /tasks/:id/heartbeat?project=X` | Query parameter: `?project=X` | Required to refresh claim lease |
| `POST /tasks/:id/logs?project=X` | Query parameter: `?project=X` | Required for appending audit logs |
| `GET /tasks/:id?project=X` | Query parameter: `?project=X` | Returns task details |
| `GET /tasks/:id` (no `?project=`) | None | Fails with `404 Not Found` |
| `GET /tasks?project=X` | Query parameter: `?project=X` | Lists tasks within project |
| `GET /projects` | No scoping needed | Global project list |

**Rule**: Append `?project=<project_name>` to every path addressing a specific task (`/:id`, `/:id/logs`, `/:id/claim`, `/:id/heartbeat`). A `403` referencing project `'default'` indicates a missing query parameter.

## 3. The Orchestration Lifecycle

The state machine is enforced via role-based transitions. The Orchestrator operates as `admin` for all status resets and transitions.

### Stage A: Feature Setup
1. **GitHub Issue**: An issue per card is **not required**. Link to an existing issue only when one already exists for that work (typically filed by the operator; e.g. #75). Do not file a new issue solely because a card was created. If no issue exists yet, leave `issues` empty and link retroactively later (`POST /tasks/:id/issues?project=X`) once one is filed.
2. **Branching**: Create `feat/<slug>` or `fix/<slug>` from `main`.
3. **Registration**: `POST /tasks` with the JSON body below. The caller MUST supply `id` (the server does not generate one) and `status`:
   ```json
   {
     "id": "<your-slug-or-id>",
     "project": "<project_name>",
     "title": "<title>",
     "description": "<what and why>",
     "branch": "<real branch from git rev-parse --abbrev-ref HEAD>",
     "issues": ["#<N>"],
     "round": 1,
     "status": "BACKLOG"
   }
   ```
   - `id`, `project`, `title`, `round`, and `status` are required.
   - Determine the active branch with `git rev-parse --abbrev-ref HEAD` and pass that exact string.
    - If a GitHub issue exists for this work, set `"issues": ["#<N>"]` in the task body so the card is structurally linked to it and will appear in the `ISSUES` swimlane. Leave `issues: []` otherwise — link retroactively when one is filed later.
   - Setting `branch` explicitly is critical for human operators and re-claim alerts.
   - `BACKLOG` and `BLOCKED` are open statuses. Creating a task directly into `BUILDING`, `IN_REVIEW`, `IN_TEST`, or `DONE` requires a privileged credential and returns `403`.
4. Record `task_id` and initial `version` from the response.

### Stage B: The Execution Cycle

**Enter an active stage by CLAIMING, never by a status-only PATCH.** `PATCH /tasks/:id` writes `status` but leaves `assigned_agent` and `claim_expires_at` unset, producing an ownerless active task that the reaper normalizes back to `BACKLOG`. `POST /tasks/:id/claim` sets the owner and lease, promoting `BACKLOG` → `BUILDING`.

1. **BUILDING**:
   - `POST /tasks/:id/claim?project=X` (`agent_id` = orchestrator id, header `X-Agent-Role: builder`). This establishes ownership, sets `claim_expires_at`, promotes `BACKLOG` → `BUILDING`, and stamps `stage_owners.BUILDING`.
   - `POST /tasks/:id/logs?project=X` (Log: "Starting build phase...").
   - Start the heartbeat loop (§4).
   - **Dispatch BUILDER**: Pass `task_id`, `repo_path`, `branch`, `kanban_url`, `kanban_token`, and `admin_token`. The builder PATCHes status and appends logs — it must NOT attempt to claim.
   - Await Builder completion and verification evidence.

2. **REVIEWING**:
   - `PATCH /tasks/:id?project=X` (Status: `IN_REVIEW`, Role: `admin`, with `expected_version`). The card is already owned from the BUILDING claim; do NOT re-claim.
   - `POST /tasks/:id/logs?project=X` (Log: "Submitting for review...").
   - **Dispatch REVIEWER**: Pass `diff` or `commit_range`.
   - Await Reviewer response (`Approve` | `Blocking Findings`).
   - If **Findings**: Append findings to logs → Transition status back to `BUILDING` → Re-dispatch Builder.

3. **TESTING**:
   - `PATCH /tasks/:id?project=X` (Status: `IN_TEST`, Role: `admin`, with `expected_version`).
   - `POST /tasks/:id/logs?project=X` (Log: "Running test suite...").
   - **Dispatch TESTER**: Pass test suite commands and verification scope.
   - Await Tester result (`Pass` | `Fail`).
   - If **Fail**:
     - `PATCH /tasks/:id?project=X` (Status: `BACKLOG`, Role: `admin`) to reset.
     - `POST /tasks/:id/logs?project=X` (Log: "Tester failed. Resetting to Backlog.").
     - Restart from step 1.

### Stage C: Closure
Validated by a `test_pass` signal:
1. **Docs & Versioning**: Dispatch worker to update README/docs/CHANGELOG and bump version numbers in lockstep.
2. **Merge**: `git checkout main && git merge --no-ff <branch>`
3. **Tag & Push**: `git tag -a v<version> -m "release <version>" && git push origin main --tags`
4. **Finalize**: `PATCH /tasks/:id?project=X` (Status: `DONE`, Role: `admin`, with `expected_version`) + log completion metadata.
5. **Close GitHub Issue**: `gh issue close <N> --comment "Resolved in v<version>: ..."`
6. **Deployment Check**: Confirm live deployment status (e.g. `railway status` and `GET /api/health`).
7. **Report**: Summarize completion to user (commit hash, test outputs, completion status).

## 4. Safety & Reliability Rules

- **Identity & Roles**: Orchestrator = `admin`. Workers = `builder`, `reviewer`, `tester`.
- **Headers**: All requests must include `x-agent-id`, `x-agent-role`, and `x-api-token`.
- **Project Scoping**: Always append `?project=<name>` on task-specific endpoints.
- **Optimistic Locking**: Every `PATCH` requires the card's CURRENT `version` as `expected_version`. Operations like `POST /claim`, `POST /logs`, and `POST /heartbeat` each increment the card's version on the server, so a version read earlier in the cycle is stale. Always fetch the fresh version before issuing a `PATCH`:
  ```bash
  GET /tasks/:id?project=X   → read `version`
  PATCH /tasks/:id?project=X → pass `expected_version`
  ```
  Do not cache `version` statically across multiple intermediate writes.
- **An unowned card reaps back to `BACKLOG`, and `BACKLOG → IN_REVIEW` is NOT a legal transition.** If you append logs without claiming, or let the lease lapse between writes, the reaper resets the card and a later status `PATCH` fails with `Invalid state transition from BACKLOG to IN_REVIEW` (HTTP 409). The fix is always the same: `POST /claim` first (which promotes `BACKLOG → BUILDING`), then `PATCH` forward. Check `assigned_agent` and `status` immediately before any transition — a card you logged to earlier in the same session can already have reaped.
- **Heartbeat & Leases**: Claim TTL is `KANBAN_CLAIM_TTL_MS` (default **600,000 ms = 10 min**; raised from 300,000 in server v2.3.11). Issue `POST /tasks/:id/heartbeat?project=X` at least **every 2 minutes** while holding a card. A claim may also request an explicit window with `{"lease_ms": <ms>}`, clamped to `KANBAN_MIN_LEASE_MS` **60,000** and `KANBAN_MAX_LEASE_MS` **7,200,000**. Logs from the holder also extend the lease.
- **Two Distinct 409 Errors** (inspect response body):
  - `"Version mismatch"` (with `details.expected` / `details.provided`): The provided `expected_version` is stale. Re-read the current version and retry the `PATCH`.
  - `"Invalid state transition from A to B"`: The lease lapsed and the background reaper reset the card to `BACKLOG`. Recover by re-claiming (`POST /claim`) and re-walking stages. Always check `git status` / `git log` on the task branch before re-dispatching, as uncommitted work may remain on disk.
- **3-Cycle Limit**: If a cycle (Build → Review → Test) repeats 3 times, halt and report blockers.
- **Evidence-Based Success**: Never transition stages without explicit command output, test results, or commit hashes.

### Reclaim Alerts (optional)

When the reaper resets a card — an expired lease (`lease_expired`) or an active card with no owner (`orphan_normalized`) — the server can POST a Telegram alert to the operator: the out-of-band companion to the lease-loss recovery path above. It is **off unless configured** (`KANBAN_TELEGRAM_BOT_TOKEN` + `KANBAN_TELEGRAM_CHAT_ID`); optional filters are `KANBAN_NOTIFY_PROJECTS`, `KANBAN_NOTIFY_EVENTS`, and `KANBAN_BOARD_URL`. The notifier is a pure subscriber — it never affects the board — so do not poll for it; treat an alert as a prompt to inspect the card and re-claim.

## 5. Data Integrity Notes

### Branch Normalization
The server normalizes branch values on write:
- Blank, empty, or whitespace-only strings (`""`, `"   "`, `"\t"`, `"\n"`) become `null`.
- Non-string values (`123`, `false`, `[]`) become `null`.
- Real refs (`fix/x`, `feat/y`) are preserved verbatim without automatic trimming.
- Do not use zero-width characters (`U+200B`).
- `branch` is patchable: update with `PATCH { "branch": "<real-branch>" }` or clear with `PATCH { "branch": null }`.

### Reading Logs Back — `agent_logs` is inline-capped and spilled

Logs are appended via `POST /tasks/:id/logs?project=X`, but the server **caps the inline array** at `KANBAN_INLINE_LOG_CAP` (default **50**) — overflow entries spill to a JSONL sidecar under `<datadir>/spill/<project>/<task-id>.jsonl` (`store.js:2435`). So long tasks can have their newest logs in the task body but older ones only on disk; verify log completeness with `GET /tasks/:id/logs?project=X&include_spilled=1`, which returns `{inline:[...], spilled_count, entries:[...]}` (`store.js:2545`). The inline array is newest-first (reverse of append order); spilled entries are oldest-first (append order). Counting `logs` in the create/append response body (or subtracting the latest version from the one on creation) confirms writes without a second fetch.

### The `ISSUES` swimlane is a cross-reference overlay, not a defect lane

A card appears in the `ISSUES` lane **only if its `issues` array is non-empty**. The lane is not a status and not a category of work: `groupTasks()` (`client/src/board-model.js`) pushes a task into `ISSUES` **in addition to** its real status lane when `task.issues.length > 0`. So a card in `DONE` with a linked issue shows in both lanes — and a card with `issues: []` is absent from `ISSUES` no matter how bug-like it is.

The array holds a **bare reference string**, conventionally `"#<N>"` for a GitHub issue in the board repo. It is not a link table and nothing validates the format — `store.addIssue` stores whatever string it is given.

**Implication for filing work:** a defect card gets a link only if you put one there. Creating a card and forgetting `issues` produces a card that is invisible in the one lane an operator scans for linked work, and nothing warns you. Set `"issues": ["#<N>"]` in the `POST /tasks` body (Stage A) so the link exists from birth.

**Owner's standing decision (30 Sep 2026): do NOT open a GitHub issue per card.** Link a card to an issue only when a genuine issue already exists for that work (typically filed by the operator, as with #75). The `ISSUES` lane is deliberately a partial index, not a complete one — the board itself is the source of truth for work in flight. Do not raise this as an open question again, and do not bulk-create issues to populate the lane. This supersedes any reading of Stage A that treats `issues` as mandatory on every card.

**Recovering a missed link after the fact** — `store.addIssue` has no status restriction, so a `DONE` card can still be linked retroactively:

```bash
POST /tasks/:id/issues?project=X   { "issue_id": "#75", "expected_version": <current> }
```

The KB-08 append takes the same §2.6 CAS guard as any other write and advances the version by exactly one, so re-read `version` first. The route returns `{ "issues": [...], "status": 200 }`; `GET /tasks/:id/issues?project=X` reads the array back. Linking after closure is normal maintenance, not a re-open — the card keeps its `DONE` status.

**When auditing a board, check `issues` explicitly rather than assuming lane absence means no issue exists.** A card can be finished, merged, and have a real GitHub issue filed for it while still carrying `issues: []`, so "not in the ISSUES lane" is not evidence that no issue was ever filed.

### Legacy Card Cleanup
Legacy cards created prior to normalization can be cleaned by:
1. `PATCH /tasks/:id?project=X` with `{"branch": null}`.
2. Sweeping cards where `branch` equals `task/<id>`.
