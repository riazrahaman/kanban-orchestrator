# Changelog

All notable changes to the `kanban-orchestrator` skill are documented here.
This project follows [Semantic Versioning](https://semver.org/).

Release tags are available on [GitHub Releases](https://github.com/riazrahaman/kanban-orchestrator/releases).

> **Version numbering.** From `2.14.3` on, the skill's version matches its
> [SkillPort listing](https://skills.syed-hasan.com/skills/riazrahaman/agentkanban).
> Earlier SkillPort versions map to the GitHub releases below:
> SkillPort `2.14.2` = `v1.0.0` (commit `09f4f90`), SkillPort `2.14.3` = `v1.2.0` (commit `cf01541`).
> `v1.1.0` was not published on SkillPort. Skill versions are independent of
> [agent-kanban-board](https://github.com/riazrahaman/agent-kanban-board) versions;
> board compatibility is listed in each entry.

## [3.9.6] — 2026-10-24

Deploy-drift awareness in the connection check, aligned with `agent-kanban-board`
v3.9.5. Ported from the board repo's bundled skill copy.

### Added

- **Deploy-drift check in the connection verification step.** On board v3.9.5+,
  `GET /api/health` also returns
  `deploy_drift: {status, running, latest_tag, checked_at, drift_since}` — the
  running build compared with the newest GitHub release. `SKILL.md` now tells the
  orchestrator how to read it:
  - `drift` means the release on GitHub is **not** the build that is live. Treat
    it as **a skipped deploy, not a failure**: a host configured to require green
    CI skips the push-triggered deployment, and the previous build keeps serving a
    healthy board. Let the operator redeploy; **do not retry writes or re-file work**
    because of drift — the board's data and API are healthy.
  - `unknown` means the check itself failed (GitHub unreachable, bad payload, or
    the monitor is disabled/just booted). It is **not** a board outage — `/api/health`
    answered, so the process is up. Report it, do not halt.
  - `in_sync` needs no action. A mismatch is only reported after a grace window
    (default 45 min), so a release that has just been tagged and not yet redeployed
    is normal, not drift.

### Changed

- **Endpoint table.** The `GET /api/health` row now documents the `deploy_drift`
  verdict (v3.9.5+) alongside the version and credential-map probes.
- **README.** Skill badge and version line bump to `v3.9.6`; the board badge and
  hero image move to `v3.9.5+`/`v3.9.6`, and the Requirements note recommends
  board v3.9.5+ for deploy-drift visibility.

### Notes

- **Version alignment.** The skill repo skipped `3.9.2`–`3.9.5` because those
  releases changed the board only. The board's bundled skill copy went
  `3.9.1` → `3.9.5` (deploy-drift section) → `3.9.6` (version line only), so this
  release republishes the skill at `3.9.6` to keep the two copies identical apart
  from the skill name and install line.


## [3.9.1] — 2026-10-20

README storefront refresh, matching the treatment on the board repo.

### Changed

- **README storefront.** Centered hero with a one-line tagline, a live badge row
  (release, license, skill version, board version, SkillPort), an anchor navigation
  row, a full-width hero image of the board this skill drives, and a `[!TIP]`
  onboarding callout that points a first-time reader straight at Install and
  Configure.
- **New "What the protocol gives you" table** stating the six guarantees the
  protocol provides — no self-approval, no lost work, no silent overwrites, no
  stage skipping, no evidence-free progress, one agent per stage — and the
  mechanism that enforces each.
- **Cleaner 8-state overview.** The lifecycle now leads with a one-line
  `BACKLOG → READY → PLANNING → IN_PROGRESS → IN_REVIEW → VALIDATION →
  READY_TO_SHIP → DONE` strip before the full dispatch flow.
- **Corrected one stale sentence.** The credential-bound roles paragraph said a
  conflicting `X-Agent-Role` header is "rejected"; the shipped behavior is that
  the header is **ignored** — the request is served with the credential's role —
  and the mismatch is audited (`role_binding_mismatch`). The wording now matches
  `SKILL.md` and the board's implementation.

### Notes

- Documentation-only change. The skill version in `SKILL.md` is bumped to
  `3.9.1` to match this release; no protocol, header, or endpoint behavior
  changed relative to `3.9.0`.

## [3.9.0] — 2026-10-20

One-agent-per-role mandate, aligned with the `role_crossing` audit signal in `agent-kanban-board` v3.9.0 (ADR-006).

### Added

- **One agent per role (MANDATORY, v3.9.0+):** dispatch each stage to a distinct `x-agent-id` **and** its own role-bound credential — never swap one agent's credential across stages to cover a card yourself. The board's opt-in `role_crossing` audit signal (`KANBAN_ROLE_CROSSING_AUDIT`, off by default) reports one agent id presenting several bound roles; it never rejects, but the correct posture is to never trigger it.

### Notes

- Board compatibility: `agent-kanban-board` v3.9.0. The signal is **off by default** — a board that has not set `KANBAN_ROLE_CROSSING_AUDIT` emits nothing, and the skill's behavior is unchanged. On a board with the signal enabled, the audit log will show `role_crossing` entries when one agent id uses several role-bound credentials.

## [3.8.0] — 2026-10-19

Credential-bound roles mandate, aligned with credential-bound authentication in `agent-kanban-board` v3.8.0 (feature branch `feat/credential-bound-roles`, not yet merged at skill-release time).

### Added
- **Credential-Bound Roles Mandate (§1, §3, §4)**: Per-role agent credentials are now **MANDATORY**: every agent role (planner, builder, reviewer, tester/validator, releaser) requires its own token bound to that role in `KANBAN_PROJECT_TOKENS`. The server derives the role from the credential; a self-declared `X-Agent-Role` header that conflicts with the credential's bound role is **ignored — the request is served with the credential's role — and the mismatch is audited** (`role_binding_mismatch`). Orchestrators use an explicit `admin` (or `runner`) credential, never a self-declared role header.
- **Role-Keyed `KANBAN_PROJECT_TOKENS` Schema (§1)**: documents the `{ "token": ..., "role": ... }` bound-credential value shape and the `{ "credentials": [...] }` array form; legacy plain-string tokens keep current unbound behavior as an opt-in migration path. Privileged roles (`admin`/`runner`/`system`/`human`) are never bound to board credentials.
- §3 updated: builder dispatch passes the builder's own role-bound token, the `READY` claim uses a builder-bound credential, and per-stage `Role:` PATCHes operate under credential-derived roles. §4 Identity & Roles and Headers rules updated accordingly.

> **Board compatibility.** The mandate targets `agent-kanban-board` v3.8.0. Boards running v3.7.3 or earlier continue to honor self-declared `X-Agent-Role` headers; role binding activates when the board upgrade ships.

## [3.7.1] — 2026-10-18

Protocol synchronization with `agent-kanban-board` v3.7.1, Ask-the-Board (Ollama and OpenRouter) public endpoints, two-stage graceful teardown architecture, strict HTTP header ByteString standards, and documentation reconciliation.

### Added
- **Ask-the-Board Public Endpoints (§2, §4, §5)**: Documented `POST /api/ask` and `GET /api/ask/config` public Q&A interfaces over board protocol and repository source context. Gated via shared password header `x-ask-pass` (`KANBAN_ASK_PASSWORD`), NOT `x-api-token`. Added support for OpenRouter and local Ollama (`KANBAN_ASK_PROVIDER=ollama`, default `qwen3.6:latest` on `http://localhost:11434`), and tunnel authentication header forwarding (`KANBAN_ASK_OLLAMA_AUTH_HEADER`). Clarified that Ask is strictly a read-only human assistant and agents must not call it for board mutations or task tracking.
- **Two-Stage Graceful Teardown (§1, §4)**: Documented board shutdown behavior where `drainSseStreams()` terminates active `/api/events` connections before listener sockets close. Documented running with `node server/server.js` (or root `npm start`) so POSIX `SIGTERM`/`SIGINT` signals propagate without wrapper process interference.
- **HTTP Header ASCII ByteString Standard (§4)**: Documented strict ByteString requirement (`U+0000` to `U+00FF`) for all HTTP headers. Non-ASCII characters (such as em dashes `—` or unicode quotes) cause client `TypeError` exceptions before packets leave the process.
- **SSE Stream Endpoint Scoping (§2)**: Added `GET /api/events` with optional `?project=` filter to the verified endpoint scoping table.

### Changed
- **Reaper Target Documentation Reconciled**: Reconciled all documentation in `README.md` and `SKILL.md` to consistently state that reaper resets, lease expirations, progress stalls, and orphan card normalizations return cards to `READY` (not `BACKLOG`), and tester failures in `VALIDATION` reset to `IN_PROGRESS` (never `READY`).
- **Board Compatibility**: Updated recommended version to `agent-kanban-board` v3.7.1+ (minimum v3.0.0+).

## [3.3.0] — 2026-10-11

Per-token project name isolation, mandatory authenticated discovery, and state machine transition integrity aligned with `agent-kanban-board` v3.3.0.

### Added
- **Per-Token Project Name Isolation (§1, §2)**: Documented tenant isolation protocol where `GET /projects` requires `x-api-token` / Bearer token authentication. In multi-tenant environments configured with `KANBAN_PROJECT_TOKENS`, callers only discover the projects explicitly assigned to their token (admin tokens retain global discovery). Unauthenticated probes receive an empty list `[]` (or `401 Unauthorized` if read authentication is enforced).
- **Scoped Health Credential Maps (§1, §2)**: Documented that `GET /api/health` credential map `missing` and `extra` arrays isolate project names strictly to the authenticated caller's authorized project scope.
- **Unscoped Task Scoping (§2)**: Documented that unscoped `GET /tasks` requests automatically scope to the caller's authorized projects rather than exposing tasks from all projects.
- **Tester Transition Rule (§3)**: Clarified that on tester failure in `VALIDATION`, cards return to `IN_PROGRESS` (`VALIDATION → READY` is not a legal state transition).

### Changed
- **Reaper Recovery Workflow (§4)**: Clarified that expired or stalled cards reset to `READY` with ownership and lease cleared, requiring workers to re-claim the card back into `IN_PROGRESS` (or `PLANNING`) from `READY`.
- **Board Compatibility**: Updated target recommendation to `agent-kanban-board` v3.3.0+.

## [3.2.2] — 2026-10-10

Mobile full-bleed background, bottom clearance reclamation, zoom restriction, and header disclosure persistence.

### Added
- **Mobile Viewport & Zoom Policy (§5)**: Documented `user-scalable=no, maximum-scale=1.0` viewport pinning and `touch-action: manipulation` across touch targets to eliminate accidental mobile zooming.
- **Full-Bleed Safari Background Protocol (§5)**: Documented `html, body` background styling and synchronized `theme-color` meta tags to ensure Safari's dynamic title bar and floating tab bar integrate seamlessly with the theme background.
- **Reclaimed Column Clearance (§5)**: Documented removal of artificial 5rem padding in board column scrolling, recovering vertical real estate above floating mobile browser chrome.
- **Header Disclosure Persistence (§5)**: Documented `kanban.headerOpen` state persistence in localStorage for narrow viewports and split-window modes.

### Changed
- **Board Compatibility**: Updated target recommendation to `agent-kanban-board` v3.2.2+.

## [3.2.1] — 2026-10-10

Reverse proxy authentication rate-limiting guidelines, strict body-parser error handling standards, and mobile UI viewport ergonomics.

### Added
- **Rate-Limiting & Proxy Trust Protocol (§4)**: Documented Express `req.ip` derivation and `KANBAN_TRUST_PROXY` proxy hop depth requirements for client IP isolation in `v3.2.1+`. Documented `429 Too Many Requests` backoff protocol respecting `Retry-After`.
- **Payload & Encoding Standards (§4)**: Documented `415 unsupported_encoding` and `400 request_size_invalid` / `request_aborted` body-parser error codes in `v3.2.1+`. Mandated uncompressed UTF-8 JSON payloads under 100KB.
- **Credential-Map Name Privacy (§1)**: Documented that `GET /api/health` withholds `missing` and `extra` project name arrays from unauthenticated probes (providing `missing_count` and `extra_count`), while authenticated callers receive complete project name lists.
- **Mobile Ergonomics & Safari Clearance (§5)**: Added reference notes for mobile viewport font sizing (13px selects, 16px text inputs) and safe-area floating bar clearance (`pb-[calc(5rem+env(safe-area-inset-bottom,0px))]`).
- **Troubleshooting Guide**: Added troubleshooting steps for `429 Too Many Requests` (auth rate limiting) and `415 Unsupported Media Type` (unsupported encoding).

### Changed
- **Board Compatibility**: Updated target recommendation to `agent-kanban-board` v3.2.1+.

## [3.1.0] — 2026-10-10

Boot-time credential-map coverage check integration and board v3.1.0 compatibility.

### Added
- **Credential-Map Coverage Check (§1)**: Updated the boot verification protocol to inspect `credential_map: {status, covered, missing, extra}` returned by `GET /api/health` on `agent-kanban-board` v3.1.0+. If the configured `project_name` is listed in `missing` or the credential map is `undercovered`/`malformed`, the orchestrator immediately alerts the operator to restore the missing token rather than encountering delayed `403 Forbidden` errors during task mutations.
- **Endpoint Reference (§2)**: Noted `credential_map` in the unscoped endpoints table for `GET /api/health`.

### Changed
- **Board Compatibility**: Declared official compatibility and recommendations for `agent-kanban-board` v3.1.0+.

## [3.0.2] — 2026-10-09

Board version check endpoint fix and documentation alignment.

### Fixed
- **Board Version Check (§1)**: Changed the version check in `SKILL.md` to query `GET /api/health` instead of `GET /projects`. `GET /api/health` returns the server's `"version"` field (supported on both v2.16.2 and v3.0.0+), whereas `GET /projects` returns an array of project summaries with no version field.
- **README Header Version**: Updated `README.md` header from `v3.0.0` to `v3.0.2`.

## [3.0.1] — 2026-10-09

### Added
- **Board Version Gate (§1)**: Added automatic version requirement check in `SKILL.md` to verify the connected board is v3.0.0+.

## [3.0.0] — 2026-10-09

Migration to AgentOS 8-state workflow lifecycle and compatibility with `agent-kanban-board` v3.0.0 (ADR-004).

### Added
- **AgentOS 8-State Workflow (§3)**: Replaced legacy 4-state pipeline with 8 canonical states: `BACKLOG → READY → PLANNING → IN_PROGRESS → IN_REVIEW → VALIDATION → READY_TO_SHIP → DONE` (+ `BLOCKED`).
- **Expanded Role Permissions (§3, §4)**: Added role-based transitions for `planner` (`READY → PLANNING`), `validator` (`VALIDATION → READY_TO_SHIP`), and `releaser` (`READY_TO_SHIP → DONE`).
- **Progress Stall Timeout (§4)**: Documented `KANBAN_PROGRESS_STALL_MS` (default 30m) reaper support alongside lease expiration.
- **Bug Reports Note (§5)**: Documented visitor bug report filing via server without unmanaged card generation.

### Changed
- **Entry into Active Stages (§3)**: Cards must be promoted to `READY` before being claimed into `IN_PROGRESS` or `PLANNING`. Claiming from `READY` establishes lease ownership.
- **Board Compatibility**: Updated target compatibility to `agent-kanban-board` v3.0.0+ (with transparent backward compatibility for v2.x status aliases).

## [2.14.7] — 2026-10-04

Reclaim and lease semantics, inline log cap with JSONL sidecar spill, and ISSUES swimlane reconciliation.

### Added
- **Reclaim & Lease Semantics (§4)**: Documented that unowned cards reap back to `BACKLOG` and `BACKLOG → IN_REVIEW` is an invalid transition requiring `POST /claim` first. Documented claim TTL (`KANBAN_CLAIM_TTL_MS`, 600,000 ms = 10 min) and heartbeat interval guidelines (every 2 minutes, lease clamped 60,000 ms to 7,200,000 ms).
- **Log Read Path (§5)**: Documented `KANBAN_INLINE_LOG_CAP=50` limit on inline task logs with JSONL sidecar spill, and complete log retrieval via `GET /tasks/:id/logs?include_spilled=1`.
- **ISSUES Swimlane Overlay (§5)**: Clarified that `ISSUES` is a cross-reference overlay (`issues.length > 0`) rather than a status lane. Documented retroactive linking via `POST /tasks/:id/issues` with CAS `expected_version`.

### Changed
- **Stage A Issue Requirement**: Reconciled Stage A to reflect the operator's standing decision (30 Sep 2026) not to file a GitHub issue per card; cards link only when an issue already exists.

## [2.14.6] — 2026-09-29

Protocol schema alignment and dynamic versioning guidance.

### Added
- **POST /tasks Required Schema**: Explicitly documented mandatory caller-supplied `id` (slug), `status: "BACKLOG"`, and `round: 1` in `SKILL.md` Stage A. Documented that work statuses (`BUILDING`, `IN_REVIEW`, etc.) require privileged roles and return `403`.
- **Dynamic Versioning & 409 Error Inspection**: Documented that `expected_version` must be re-read before each `PATCH` because intermediate `claim`, `logs`, and `heartbeat` calls increment card versions on the server. Added distinct handling for `409 Version mismatch` (retry with fresh version) vs `409 Invalid state transition` (lease expired and reaped; re-claim card).

### Fixed
- **Plural Install Path**: Corrected install paths in `SKILL.md` and `README.md` to `.opencode/skills/kanban-orchestrator/` (plural).

## [2.14.4] — 2026-09-28

Human-facing overview and docs structure.

### Changed
- Rewrote `README.md` as human-facing overview with visual architecture diagrams and clear stage workflows.

## [2.14.3] — 2026-09-28

Version renumbering only. The protocol is unchanged from `1.2.0`.

### Changed
- Skill frontmatter `version` set to `2.14.3` to match the SkillPort listing.
- Compatible with upstream `agent-kanban-board` `v2.14.0` through `v2.15.1+`.

## [1.2.0] — 2026-09-28

Protocol alignment with [agent-kanban-board](https://github.com/riazrahaman/agent-kanban-board) v2.15.1.

### Added
- **GitHub Issue Lifecycle**: Integrated formal GitHub issue tracking into Stage A (issue creation or linking via `"issues": ["#<N>"]`) and Stage C (issue resolution and closure via `gh issue close`).
- **Release Tagging & Deployment Verification**: Added step-by-step release tagging (`git tag -a`) and production deployment verification (`railway status` and health checks) to Stage C closure protocol.
- **Lockstep Documentation & Versioning**: Formalized lockstep version bumping across README, docs, and CHANGELOG.

### Changed
- Updated skill frontmatter version to `1.2.0`.
- Documented compatibility with upstream `agent-kanban-board` through `v2.15.1+`.

## [1.1.0] — 2026-09-27

Protocol alignment with [agent-kanban-board](https://github.com/riazrahaman/agent-kanban-board) v2.14.3.

### Added
- **Changelog**: Introduced `CHANGELOG.md` for tracking skill protocol and documentation updates.
- **Compatibility Documentation**: Documented compatibility with upstream `agent-kanban-board` v2.14.0+ through v2.14.3+.

### Changed
- Aligned skill protocol documentation with upstream `v2.14.3`, including branch normalization rules and Telegram reclaim alert mechanisms.
- Updated `README.md` with links to changelog and upstream board releases.

## [1.0.0] — 2026-09-26

Initial standalone release of the `kanban-orchestrator` skill.

### Added
- **Protocol Specification**: Standalone `SKILL.md` orchestrator protocol for OpenCode coding agents driving `agent-kanban-board`.
- **Automatic Local Deployment**: Automated local clone, build, and run instructions if no remote board URL is configured.
- **Role-based Lifecycle**: Step-by-step workflow covering claim-first `BUILDING`, `IN_REVIEW`, `IN_TEST`, and `DONE` stages.
- **Safety Rules**: Enforces optimistic locking with `expected_version`, heartbeats, and per-claim lease windows.
