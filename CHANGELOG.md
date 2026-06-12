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
