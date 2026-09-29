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

## [2.14.5] — 2026-09-29

Protocol fixes found by checking `SKILL.md` against the agent-kanban-board server source, plus claude.ai upload compatibility.

### Fixed
- **Task registration sends `id`**: `POST /tasks` rejects a body without a caller-chosen `id` (`400 id and title are required`). Stage A now requires one and documents the allowed charset (`[A-Za-z0-9_-]`; a branch name with `/` is not a valid id) and the `409 Task <project>/<id> already exists` recovery.
- **`kanban_url` includes `/api`**: every board route lives under `/api`; without it `GET /projects` returns the web UI's HTML with a `200`. The config section and local deployment step 4 now say so.
- **Verification checks URL and token separately**: `GET /projects` is readable without a token by default and lists a project only after its first card, so it can't verify the token or project existence. The first `POST /tasks` is now the token check (`201` / `401` / `503`).
- **Local deployment sets a token**: the board refuses writes with `503` until `KANBAN_AUTH_TOKEN` is set and doesn't read `.env`. Step 3 now exports it before `npm start`, and step 5 reuses it as `kanban_token`.
- **SkillPort install path** (README): the CLI installs to `.claude/skills/riazrahaman__agentkanban/` (or `./skills/riazrahaman__agentkanban/` with `--target generic`), not `./skills/agentkanban/`.

### Changed
- Frontmatter `version` moved to `metadata.version`. A top-level `version` key fails claude.ai's skill upload validation (allowed keys: `name`, `description`, `license`, `allowed-tools`, `metadata`, `compatibility`). The value drops the `v` prefix to stay semver; tags keep `v<version>`.
- Compatible with upstream `agent-kanban-board` `v2.14.0` through `v2.15.1+`.

## [2.14.4] — 2026-09-28

Documentation only. The protocol is unchanged from `2.14.3`.

### Changed
- README rewritten as a human-facing overview.

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
