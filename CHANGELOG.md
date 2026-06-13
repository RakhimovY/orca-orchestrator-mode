# Changelog

## 0.7.0 - 2026-06-14

Borrow pass from a deep-dive of two popular frameworks (crewAI ~54k stars, Task Master ~27k stars). A RED baseline probe first established the skill was already strong: quality-under-cost-pressure, ready-set sequencing, and acceptance criteria all PASSED at baseline, so those were NOT re-added. Three genuine gaps + five user-requested capability borrows shipped; all GREEN-verified on one consolidated 8-trap scenario, zero loopholes, no REFACTOR. Probe log: `docs/probe-log-2026-06-14.md`. Source analysis: `references/crewai.md`, `references/claude-task-master.md` (private KB).

Probe-driven (RED gap → GREEN):

- **Task contract = typed edges, not prose (new section).** RED: the orchestrator dispatched with free-text acceptance and accepted a prose `DONE:` line, capturing no predecessor decisions. New: every task carries `done_when` + a `verify:` command the worker must pass before `worker_done`; `verify` red → self-correct (bounded attempts) before escalating; `worker_done` is a STRUCTURED payload `{summary, files_changed, verify, key_decisions, follow_ups, blockers}`. The coordinator decides FROM the payload, not the diff. (crewAI TaskOutput + Task Master testStrategy.)
- **Dependency graph: validate + ready-set + handoff (§ Conflicts).** RED: handoff only worked when the predecessor's decision was spoon-fed into the scenario; no graph validation. New: validate the graph (no cycles/dangling) and dispatch ONLY from the ready-set (hard-deps Done + no lease conflict + free zones, recomputed on each `worker_done`); a dependent worker's prompt MUST carry the predecessor's `key_decisions[]` + summary. (Task Master next/dependencies + crewAI Task.context.)
- **Envelope check-in for active over-runs (§ Reading workers).** RED: liveness covers only SILENT workers; an active worker 3x past estimate defaulted to "let it cook". New: a non-killing checkpoint demand when an active worker runs far past its effort estimate (the rabbit-hole signal liveness misses). (crewAI max_iter/max_execution_time, reframed as a quality guard, not a cost cap.)

User-requested capability borrows (real payoff, not RED-driven; GREEN-verified on recognition scenarios):

- **Optimize for correctness, not token-thrift (§ Models & budget, lead principle).** Makes the already-emergent behavior explicit and scenario-general: never drop a verify/review pass, downgrade a code tier, or skip warranted research to save tokens; economize only on mechanics. The budget guard informs, never self-restricts quality. (User-directed; baseline already GREEN, this hardens it.)
- **Queue-draining worker (matrix).** A batch of small, same-zone, non-conflicting, sequential tasks → one long-lived drainer (next→do→verify→done→next) instead of N worktrees + N PRs; trade-off (shared branch, serial) stated. (Task Master loop.)
- **Pre-dispatch prep (§ Conflicts):** an effort estimate (feeds the envelope), a research pre-pass for high-uncertainty tasks, and a one-time spec/PRD → draft tracker issues seeding for approval - never a persistent parallel task file (dual-store drift is exactly the Task Master anti-pattern we reject; the tracker stays canonical). (Task Master analyze-complexity / research / parse-prd.)
- **Reconcile downstream on drift (§ PR flow).** A merged PR's `key_decisions[]` that invalidate a not-yet-started downstream task → update the task before dispatching its worker. (Task Master update --from.)

Deliberately NOT borrowed: persistent `tasks.json` (a second source of truth vs the tracker), tagged task lists (the tracker handles contexts), crewAI hierarchical manager (our adaptive check-wait loop is stronger), crewAI memory tiers (KB + tracker + git ARE our memory), cost/trace observability dashboards (user is not token-constrained; pure telemetry, no quality payoff).

## 0.6.2 - 2026-06-13

Knowledge-correction from a live RED (silent decision-gate deadlock).

- **worker-mode: a blocking signal must be EMITTED to the bus, not just recorded in worktree status.** A worker hit an architecture fork, wrote it to its worktree status/comment, and ended its turn expecting the coordinator's `check` loop to "see the decision_gate" - but never emitted one. Silent deadlock (coordinator's bus empty, worker waiting forever) until the user noticed the stall. New rule (§ Reporting): emit `decision_gate`/`ask`/`BLOCKED` as a real bus message THEN wait; "end your turn cleanly" applies only AFTER the signal is in flight; a block recorded only in worktree status/commits is invisible to the coordinator.
- **orchestrator-mode: the monitor is the ACTIVE check-wait loop + a worktree-status glance, never passive push-trust.** Same incident: the coordinator went idle "trusting push-injection" instead of running the prescribed loop, and a worker can block without any bus message anyway. New rule (§ Reading workers): bus-silent is NOT "working" - at each wait window also glance the worker's worktree git-status/new-commits/transcript/process; a silent worker with no commits + idle process may be blocked-and-waiting.

## 0.6.1 - 2026-06-13

Knowledge-correction from a live RED (silent model-init hang).

- **Launch-health rule (§ Reading workers, both editions).** A worker launched with custom `--model`/`--effort`/`--permission-mode` can die at model-init: an unavailable model or a bad flag fails at the CLI/API layer BEFORE the agent runs, so it never starts, cannot post `escalation`/`BLOCKED`, and the bus stays silent forever. New rule: confirm launch health ~30-60s after send (tab title → task-derived summary + active spinner = alive; frozen spinner / model error / shell with a resume hint = dead), NEVER trust the first frame (it only shows prompt-ingestion) and NEVER trust the terminal-list `preview` (it lags with stale scrollback) - the definitive check is `ps` on the process to read its actual `--model` arg. RED 2026-06-13: a worker was launched on a model the user had said was unavailable; it hung mute and a first-frame liveness check passed falsely.
- Private edition binding only: the concrete model matrix moved off the (currently unavailable) Fable 5 onto Opus 4.8 1M for the Orchestrator + hard/security tier, with a note that the `[1m]` model-id must be shell-quoted (zsh globs `[1m]`). The portable edition's matrix is model-agnostic ([bracketed] tiers) so it is unchanged - a binding-only correction by definition lives in the bound edition.

## 0.6.0 - 2026-06-13

Comms-channel correction (supersedes 0.5.0's "org bus is the default for any tracked worker").

- **Mode-dependent channel choice.** Root cause traced and verified: the orchestrator pane runs inside an Orca-managed terminal whose agent-hook (Stop/UserPromptSubmit/PostToolUse) reports lifecycle to the local runtime, which then PUSH-injects pending org-bus messages into the pane as turns. Confirmed live by the pane's `ORCA_PANE_KEY`/`ORCA_AGENT_HOOK_PORT` env. This push is exactly right for an autonomous wave (the orchestrator is woken to dispatch the next thing; full tracking/DAG/ask-reply) but barges into a LIVE user conversation in the same pane. New rule: **file-bus while the user is co-working live in the panel** (passive, polled, summaries only - zero injects), **org bus for autonomous waves when the user has stepped away**. The 0.5.0 org-bus message-discipline (workers emit only worker_done/escalation/decision_gate/BLOCKED) still applies whenever the org bus is in use.
- Diagnostic note for future-self: when "why are worker messages appearing in my chat" recurs, it is the Orca agent-hook push, not a skill-rule change - the comms rule has been org-bus-capable since 0.1.0; the variable is the transport actually chosen and whether the user shares the pane.

## 0.5.0 - 2026-06-13

Knowledge-correction pass from a live wave (user-gated; all decisions confirmed in-session).

- **Worker permission default flipped to full bypass.** RED: the `acceptEdits + allowlist` default made autonomous infra workers hang INVISIBLY on un-allowlisted commands, and the list was dead weight to maintain. Autonomous isolated-worktree workers now default to full bypass (no allowlist); `acceptEdits + allowlist` is reserved for visible/interactive sessions with a deliberate human gate.
- **Anti-nesting + destructive-op guards moved into the worker prompt.** Bypass ignores allow/deny lists, so the old "physical" allowlist-deny (no worktree/terminal create, no push-to-main, no prod deploy, no `rm -rf` outside) no longer exists - it is now a mandatory forbidden-commands clause in every worker prompt, backed by the role line and the coordinator-decomposes rule.
- **Worker message discipline (inject fix).** RED: on the org bus, worker heartbeats and routine STATUS pings inject into the live orchestrator session and interrupt the user mid-conversation. Workers now emit ONLY `worker_done` / `escalation` / `decision_gate`; silence between milestones. Org bus stays the default for tracked workers.
- **File-bus wake caveat.** RED: a file-bus worker that ends a turn goes idle and does not auto-poll - an appended `ANSWER` won't wake it, a `terminal send` poke may still not re-engage. Documented; reinforces org bus as default (`reply` auto-delivers/wakes).
- **Teardown ritual before worktree rm.** RED: worktrees removed with the tracker closed but the day-log never written - narrative drift. Orchestrator now writes the worker's outcome to the day-log (as events happen) and closes the tracker BEFORE rm.

## 0.4.0 - 2026-06-13

Consolidation release. The skill family accreted fast over two days; this pass pays the discipline debt.

- **Language unification of the upstream private editions.** The private (bound) editions of all three skills were rewritten in pure English, semantically identical rule-by-rule to their pre-translation state (verified by an independent comparison pass). The portable editions in this repo and the private editions now diff in bindings only ([bracketed] tiers/KB/tracker here vs concrete models/vault/Linear there).
- **Retroactive GREEN probe suite.** Seven rules that shipped in 0.2.0-0.3.1 from live REDs but without a GREEN replay got one each: one-terminal spawn, terminal-neighbor write-access boundary, session-audit duty, codex provisioning + limit gate, boundary compaction, browser routing, and the pr-review public-repo rule. 9/9 PASS (incl. the two new rules below), zero loopholes, no REFACTOR needed. Full log: `docs/probe-log-2026-06-13.md`.
- **worker-mode: post-worker_done turn end.** Live RED 2026-06-13: a worker, after sending worker_done, invented a grace-period polling loop to wait for follow-ups, dripping tokens and masking completion. New rule: after worker_done, END YOUR TURN cleanly - no polling loops, no hold-open waiting; an idle session at the prompt costs zero tokens and stays fully reachable for a re-dispatch or terminal send. Explicitly overrides generic grace-period instructions in dispatch-preamble boilerplate. Plus a red-flags row.
- **Skill lifecycle step 7: skill-edit discipline.** RED = this very day (rules added without probes, an OSS mirror commit shipped incomplete). Every skill edit ships in the same sitting as either its probe (RED/trap replayed to GREEN) or an explicit knowledge-correction tag citing the verified observation; two editions of a skill are edited as a PAIR in one sitting - a one-edition edit is an unfinished edit, regardless of size. No third path. Plus an excuse-table row.
- Minor parity fixes between editions: execution-matrix verdict marked "in the user's language"; compaction offer threshold normalized to >20%; `tui-idle` noted as a heuristic with false positives in both editions.
- Changelog reordered to newest-first (0.3.1 had been inserted out of order).

## 0.3.1 - 2026-06-13

- **One worker = one terminal.** Live failure: spawning with custom agent flags (worktree create without `--agent`, then a NEW `terminal create --command claude ...`) leaves the worktree's auto-created default shell dangling next to the agent. Rule: launch the agent INSIDE the auto-created first terminal via `terminal send`; close stray panes immediately.

## 0.3.0 - 2026-06-12

- **Two role skills below the orchestrator: `skills/worker-mode` and `skills/pr-review`.** Built TDD-style against a live failure corpus from a real 12-phase build day: a worker peeking into the orchestrator skill for its routing table (so worker-mode duplicates the routing and bans the peek), plan checkboxes left unticked degrading post-compaction recovery (boundary protocol: tick, commit, STATUS, `BOUNDARY: ready for /compact` marker - the coordinator triggers compaction, never auto-compaction mid-task), a launch profile silently overriding the declared permission mode (mandatory verify-and-report at session start), and an independent reviewer finding 2 BLOCKERs after the author self-reported "security clean" (pr-review: the author's self-report is not verification; checklist with blanket-GRANT, caller-supplied tenant id and per-platform escaping as named classes; findings without file:line don't exist; no posting vulnerabilities to public PRs before fixes). Both passed 6-trap GREEN probes. Orchestrator SKILL.md gains a role-skill registry; per-role policy lives in the role's skill only.

## 0.2.0 - 2026-06-12

- **Browser routing.** Live-verified: Orca's built-in browser has a disk-persistent partition SHARED across all worktrees (log in once, the fleet has it, survives restarts) and device emulation via `set device` - making it the default surface for worker QA/UI tasks with no exclusive lease needed. The real desktop browser stays exclusive and user-profile-only; its permission click is a human gate the orchestrator never self-approves. Logins are typed by the user only.
- **Idle backlog scan.** Baseline probe (live day with one long-running build worker): the orchestrator handles liveness and hygiene at empty wait windows, but never looks at the backlog - long waits are dead time, and nothing prevents proposing the same declined task every cycle. New section: at wait checkpoints (never on a timer), scan the current product's open tracker tasks + the checkpoint's ready-next list; self-serve only ready-next items within the matrix; new candidates = one-line proposal (task, env, cost, conflict verdict) behind the user's yes; an anti-nag ledger in the wave checkpoint (candidate → verdict → reason) with re-offers only on that candidate's state delta; new-task proposals only from verified observations, tracker-created after the user's yes. Empty scan = silence. Plus two new excuse-table rows. Verified GREEN on a 3-trap scenario (declined-no-delta silence, new-task propose-not-take, ready-next self-serve).

## 0.1.1 - 2026-06-12

- **Nesting rule (fleet depth = 1).** Baseline probe showed a dispatched worker, when granted "sub-orchestrate yourself" by its coordinator, happily spawns its own worktree workers - bypassing the human budget gate and visibility. New rules: workers never create worktrees/terminals or invoke orchestrator skills; a worker needing siblings asks, and the coordinator spawns them itself; sub-coordinators only by explicit user grant; mandatory role line in every worker prompt + repo-allowlist deny as physical enforcement.

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
