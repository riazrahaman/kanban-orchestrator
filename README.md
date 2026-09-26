# kanban-orchestrator

A [opencode](https://opencode.ai) skill that turns a coding agent into a **strict
Kanban-first orchestrator** for delegated builds, tasks, and feature workflows.
It drives an [agent-kanban-board](https://github.com/riazrahaman/agent-kanban-board)
board as the single source of truth: you hold the lease, dispatch builder /
reviewer / tester workers, and move each card through the state machine with
evidence.

## What it does

- **Board is the state machine.** Every task lives on a card; stage transitions
  are role-gated and validated server-side. Illegal moves return `409`, wrong
  roles return `403`.
- **Claim to own, PATCH to move.** Entering an active stage means claiming the
  card (`POST /tasks/:id/claim`), which sets the owner and a lease. A
  status-only PATCH produces an ownerless card the reaper resets to `BACKLOG`.
- **Leases beat crashes.** A held card is kept alive with heartbeats; if the
  holder disappears the reaper returns the card to `BACKLOG` so another agent
  can pick it up. Uncommitted work is never lost, but it is never committed
  either — always inspect the branch after a reclaim.
- **Scoped by project.** Every task path carries `?project=<name>`; only
  `POST /tasks` takes the project in the body.

## Install

Copy the `kanban-orchestrator/` directory into your opencode skill path:

```bash
# project-local
cp -r kanban-orchestrator .opencode/skill/kanban-orchestrator

# or global
cp -r kanban-orchestrator ~/.config/opencode/skill/kanban-orchestrator
```

The directory name must match the `name:` field in the skill's frontmatter
(`kanban-orchestrator`).

## Configure

The skill needs three mandatory fields — `kanban_url`, `kanban_token`, and
`project_name` — resolved from:

1. `.opencode/config.json` (also accepts `admin_token`), or
2. environment variables `KANBAN_URL`, `KANBAN_TOKEN`, `KANBAN_PROJECT`,
   `KANBAN_ADMIN_TOKEN`.

Never commit `.opencode/` — it holds tokens.

If you don't have a board yet, the skill can deploy one locally: it clones
[agent-kanban-board](https://github.com/riazrahaman/agent-kanban-board),
installs each package, builds the client, and starts the server (default
`http://localhost:4000`).

## The protocol

1. **Set up** — branch `feat/<slug>` or `fix/<slug>`, register the card with the
   real branch name, record its id and version.
2. **Build** — claim the card, dispatch the builder with evidence, await
   completion.
3. **Review** — move to `IN_REVIEW`, dispatch the reviewer; findings loop back
   to building.
4. **Test** — move to `IN_TEST`, dispatch the tester; a fail resets to
   `BACKLOG`.
5. **Close** — update docs, merge with `--no-ff`, push, mark `DONE`, report.

A full cycle that repeats three times must halt and report blockers.

## Safety highlights

- Optimistic locking: every `PATCH` sends `expected_version` (stale writes get
  `409`).
- Heartbeat at least every `KANBAN_CLAIM_TTL_MS / 2` (default TTL is
  600,000 ms = 10 min).
- Optional per-claim lease window via `{"lease_ms": <ms>}`, clamped to
  `KANBAN_MIN_LEASE_MS` 60,000 / `KANBAN_MAX_LEASE_MS` 7,200,000.
- Optional Telegram reclaim alerts when the reaper resets a card (off unless
  `KANBAN_TELEGRAM_BOT_TOKEN` + `KANBAN_TELEGRAM_CHAT_ID` are set).

## License

MIT — see [LICENSE](LICENSE).
