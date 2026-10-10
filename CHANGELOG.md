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
