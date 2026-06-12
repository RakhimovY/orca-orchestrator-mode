# Changelog

## 0.1.0 - 2026-06-12

Initial public release ("portable edition" of a private, vault-wired skill).

- Execution matrix (in-session / subagent / Orca terminal / worktree) + branch and split rules, MAX_PARALLEL_WORKERS=4
- Tracked dispatch sequence over Orca orchestration bus, incl. live-verified gotchas: manual `task-update` after `worker_done`; disposable-worker cleanup via `worktree rm` (not `terminal close`)
- Worker reading rules (no TUI reading; bus + artifacts; liveness pings; dead-worker procedure)
- Model/budget routing framework with second-pool (Codex) quota guards
- Worker permissions: acceptEdits + curated allowlist template, deny-list, codex sandbox flags
- Conflict pre-flight, exclusive-resource leases, tracker conventions (`## Exec` block, closing discipline)
- PR flow: sequential reviewer sessions, user merge gate, rebase via `--force-with-lease` to own branch
- Orchestrator context lifecycle: wave checkpoint schema, restart procedure, 70%/85% compaction rules
- Knowledge-correction duty; human-gated skill lifecycle (anti skill-bloat, TDD authoring)

## 0.1.1 - 2026-06-12

- **Nesting rule (fleet depth = 1).** Baseline probe showed a dispatched worker, when granted "sub-orchestrate yourself" by its coordinator, happily spawns its own worktree workers - bypassing the human budget gate and visibility. New rules: workers never create worktrees/terminals or invoke orchestrator skills; a worker needing siblings asks, and the coordinator spawns them itself; sub-coordinators only by explicit user grant; mandatory role line in every worker prompt + repo-allowlist deny as physical enforcement.

## 0.3.0 - 2026-06-12

- **Two role skills below the orchestrator: `skills/worker-mode` and `skills/pr-review`.** Built TDD-style against a live failure corpus from a real 12-phase build day: a worker peeking into the orchestrator skill for its routing table (so worker-mode duplicates the routing and bans the peek), plan checkboxes left unticked degrading post-compaction recovery (boundary protocol: tick, commit, STATUS, `BOUNDARY: ready for /compact` marker - the coordinator triggers compaction, never auto-compaction mid-task), a launch profile silently overriding the declared permission mode (mandatory verify-and-report at session start), and an independent reviewer finding 2 BLOCKERs after the author self-reported "security clean" (pr-review: the author's self-report is not verification; checklist with blanket-GRANT, caller-supplied tenant id and per-platform escaping as named classes; findings without file:line don't exist; no posting vulnerabilities to public PRs before fixes). Both passed 6-trap GREEN probes. Orchestrator SKILL.md gains a role-skill registry; per-role policy lives in the role's skill only.

## 0.2.0 - 2026-06-12

- **Browser routing.** Live-verified: Orca's built-in browser has a disk-persistent partition SHARED across all worktrees (log in once, the fleet has it, survives restarts) and device emulation via `set device` - making it the default surface for worker QA/UI tasks with no exclusive lease needed. The real desktop browser stays exclusive and user-profile-only; its permission click is a human gate the orchestrator never self-approves. Logins are typed by the user only.
- **Idle backlog scan.** Baseline probe (live day with one long-running build worker): the orchestrator handles liveness and hygiene at empty wait windows, but never looks at the backlog - long waits are dead time, and nothing prevents proposing the same declined task every cycle. New section: at wait checkpoints (never on a timer), scan the current product's open tracker tasks + the checkpoint's ready-next list; self-serve only ready-next items within the matrix; new candidates = one-line proposal (task, env, cost, conflict verdict) behind the user's yes; an anti-nag ledger in the wave checkpoint (candidate → verdict → reason) with re-offers only on that candidate's state delta; new-task proposals only from verified observations, tracker-created after the user's yes. Empty scan = silence. Plus two new excuse-table rows. Verified GREEN on a 3-trap scenario (declined-no-delta silence, new-task propose-not-take, ready-next self-serve).
