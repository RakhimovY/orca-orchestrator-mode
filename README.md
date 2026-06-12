# orca-orchestrator-mode

An orchestration **policy skill** for running fleets of coding agents (Claude Code, Codex CLI) inside the [Orca IDE](https://onorca.dev) - one main session acts as a team-lead orchestrator, workers run in isolated git worktrees, coordination goes through Orca's orchestration bus.

Built and battle-tested while shipping a real SaaS with 3+ agent sessions running in parallel (build agent in a worktree, infra agent in a browser-driving terminal, orchestrator supervising both). Every rule in the skill exists because something actually went wrong without it.

## What problem this solves

Running multiple coding agents in parallel fails in predictable ways:

- **You can't see what an agent is doing from outside.** Agent CLIs draw in the terminal's alternate screen buffer - `terminal read` returns a spinner. If you didn't set up a communication channel BEFORE launching the session, you are blind.
- **Workers stall invisibly on permission prompts.** A worker stuck on "Allow this tool?" looks identical to a worker thinking hard.
- **Parallel agents trample each other** - two writers in one checkout, two sessions fighting over one Chrome, a "quick fix on main" colliding with a feature branch.
- **Knowledge rots**: yesterday's "how this works" note is wrong today, and the agent confidently reuses it.
- **Skill libraries bloat**: self-creating-skill agents (GenericAgent-style) accumulate hundreds of untested half-right skills.

This skill encodes the counter-policies: an execution matrix (where each piece of work runs), a tracked dispatch protocol over Orca's orchestration bus, worker permission templates, model/budget routing, a PR flow with a human merge gate, reporting discipline, and a human-gated self-improvement loop.

## What's in the box

- **`skills/worker-mode/SKILL.md`** - the worker-side policy: subagent-driven execution, model routing for the worker's own subagents, the phase-boundary protocol (tick plan checkboxes, commit, STATUS, signal readiness for coordinator-triggered compaction), bus reporting, the post-worker_done turn-end rule (no grace-period polling loops - an idle session is free and reachable), PR authoring rules, QA discipline in Orca's built-in browser, and the anti-matryoshka role line. Embed "invoke worker-mode first" in every worker prompt.
- **`skills/pr-review/SKILL.md`** - the reviewer-side policy: read-only (no checkout), contract-first, sequential code + security passes with a concrete checklist (blanket GRANTs, caller-supplied tenant ids, per-platform escaping), self-verification of every blocker, verdict-file output, and no posting of vulnerabilities to public PRs before fixes.
- **`SKILL.md`** - the orchestrator policy. Install it for your agent (see below). Sections:
  - Execution matrix: in-session vs invisible subagent vs visible Orca terminal vs worktree worker
  - Tracked worker spawning (the exact 6-step dispatch sequence + the gotchas we hit live: manual `task-update` after `worker_done`, `worktree rm` not `terminal close` for cleanup)
  - Reading workers without reading their TUI (bus + artifacts only, liveness rules, dead-worker procedure)
  - Model & budget routing framework (frontier for code, fast tiers for mechanics, cheap models for research, second pool with quota guards)
  - Worker permissions (`acceptEdits` + curated allowlist, deny-list for destructive ops)
  - Conflict pre-flight (exclusive resources, file zones, dependency chains) and issue-tracker conventions
  - PR flow with sequential review sessions and a user-held merge gate
  - Context lifecycle for the orchestrator (wave checkpoints, restart procedure, emergency compaction)
  - Knowledge-correction duty and a human-gated skill lifecycle (incl. skill-edit discipline: every edit ships with its probe or a knowledge-correction tag, both editions in one sitting)

## Requirements

- [Orca](https://onorca.dev) with the **orchestration** experimental feature enabled (Settings → Experimental), and its native skills `orca-cli` + `orchestration` available to your agent. This skill is the policy layer ON TOP of those - it does not duplicate their mechanics and will tell the agent to read them.
- A coding agent CLI: built for **Claude Code**, with **Codex CLI** as an optional second pool.
- **Wiring (important):** the skill assumes a persistent **knowledge base** (Obsidian-style vault / docs repo) and an **issue tracker** (Linear, GitHub Issues). Placeholders in `[brackets]` in SKILL.md mark where to bind yours. Orchestration without persistent memory degrades fast: the orchestrator's context dies daily and workers are stateless - the KB and tracker ARE the memory.

## Install

```bash
# Claude Code (personal skills)
git clone https://github.com/RakhimovY/orca-orchestrator-mode.git
mkdir -p ~/.claude/skills/orca-orchestrator-mode
cp orca-orchestrator-mode/SKILL.md ~/.claude/skills/orca-orchestrator-mode/SKILL.md
```

Then edit the `[bracketed]` bindings (your models, KB paths, tracker) and add a trigger line to your global `CLAUDE.md`, e.g.:

> Invoke the `orca-orchestrator-mode` skill BEFORE launching any worktree worker or 2+ subagents in one wave, taking a tracker task into work, or using an exclusive resource (browser/emulator/prod DB) while other sessions run.

## Field notes (why some rules look paranoid)

- `terminal read` on a Claude Code worker returns only a spinner - we learned this with two sessions already in flight and had to retrofit a file-bus protocol. Hence: *channel before launch, no retrofit*.
- A smoke run showed `worker_done` does NOT auto-complete the orchestration task - it dangles as `dispatched` until the coordinator runs `task-update`. In the skill.
- Closing a disposable worker's last terminal pane auto-spawns a fresh empty agent session and the workspace lingers ("why is there an idle Claude Code here?"). Cleanup = `worktree rm`. In the skill.
- A reviewer agent confidently declared a current model ID "hallucinated" based on its stale training data. Hence: *verify mechanics against `--help`/docs at execution time, never from memory* - the rule applies to reviewers too.

## Authoring methodology

The skill was written with TDD-for-documentation: a baseline scenario was run WITHOUT the skill (the agent chose to read worker TUIs, retrofit comms, push to main next to a live worker, and launch Codex without sandbox flags), the skill was written against those exact failures, the same scenario was re-run WITH the skill (clean), and a live smoke test exercised the full tracked dispatch cycle. Post-release edits follow the same loop: a real observed failure is the RED test. Probe transcripts for releases live under `docs/` (e.g. `docs/probe-log-2026-06-13.md` for 0.4.0).

## License

MIT
