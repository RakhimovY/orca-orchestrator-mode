---
name: orca-orchestrator-mode
description: Use when launching any worktree worker or 2+ subagents in one wave, taking an issue-tracker task into work, needing an exclusive resource (Chrome, emulator, prod DB) while other sessions run, or when the user asks to run/parallelize/orchestrate work. Also use before deciding WHERE a task should execute (in-session vs subagent vs Orca terminal vs worktree).
---

# Orca Orchestrator Mode

Policy for a main agent session (the orchestrator) coordinating worker agent sessions inside the [Orca IDE](https://onorca.dev). Mechanics live in Orca's native skills `orca-cli` and `orchestration` (Orca-managed - never edit them). This skill decides WHAT/WHERE/WHO; read those for HOW.

**Never invent mechanics.** Verify claude/codex/orca flags against `--help` or current docs at execution time, not from memory.

> **Wiring required.** This policy assumes you have: (1) a persistent knowledge base the orchestrator reads/writes (an Obsidian-style vault, a docs repo - referred to as **KB** below); (2) an issue tracker (Linear, GitHub Issues - referred to as **tracker**); (3) your own model subscriptions. Replace the placeholders in [brackets] with your bindings. The skill degrades without them: workers are stateless and the orchestrator's context dies daily - KB + tracker ARE the memory.

## Execution matrix (decide first, tell user one line: `Matrix: <env>, <reason>`)

| Env | When | Never |
|---|---|---|
| in-session | <15 min, no repo code, needs orchestrator context | long/heavy work that bloats orchestrator context |
| subagent (invisible) | read-only / isolated output, user need not watch, often N of them | anything needing browser-login, secrets entry, or user participation |
| Orca terminal in existing checkout (visible) | interactive + external services/browser; no repo code (or atomic main change) | code while parallel tasks run on the repo |
| Orca worktree (visible, own branch) | writes repo code; hours; parallel; ends in PR | - |

- Direct-to-main code: ONLY if no parallel sessions on that repo AND atomic (1-2 files, can't break build). A running worker on the repo = use a worktree even for a CSS hotfix.
- Visibility escalation = kill+respawn: a subagent that turns out to need a browser/secrets/user returns partial results + a handoff summary; spawn a fresh visible session with that summary in the prompt.
- Splits inside one worktree: helper roles only (reviewer, test watcher, dev server). Never a second writing agent.
- **MAX_PARALLEL_WORKERS = 4** worktree workers. More = explicit user override.
- **Nesting: fleet depth = 1, always.** Workers freely use invisible subagents INSIDE their session, but NEVER create worktrees/terminals or invoke this skill. A worker needing siblings sends `ask`; the coordinator NEVER replies "sub-orchestrate yourself" (baseline-tested: a worker happily builds a matryoshka on such a grant, bypassing the user's budget gate) - the coordinator decomposes and spawns the siblings itself, under the global cap and the user's visibility. A sub-coordinator exists only by explicit USER grant (full handoff). Enforcement: the role line in every worker prompt (below) + deny `orca worktree create`/`orca terminal create` in the repo allowlist (workers live in the repo, the orchestrator doesn't - the ban is physical).

## Spawning workers (tracked = default for any real task)

Exact sequence (details in the `orchestration` skill):
1. `orca worktree create --repo <sel> --name <task> --agent <claude|codex> --json` (NO --prompt) → worktreeId. `--agent` launches the agent IN the workspace's first terminal - no separate terminal create needed.
   **Custom agent flags (--model/--permission-mode/--allowedTools):** `--agent` doesn't take them → create the worktree WITHOUT --agent, take its auto-created first terminal from `terminal list`, and launch the agent INSIDE it via `terminal send --text 'claude --model ...' --enter`. Do NOT create a second terminal next to the empty default one (live failure: a workspace with a dangling zsh + the agent). Rule: **one WRITING agent per worktree, no empty panes left behind**. Additional terminals in the same worktree are allowed and don't conflict technically (each is its own PTY, dispatch targets a specific handle) - but helper roles only (a reviewer of this same diff, a test watcher, a dev server). A second agent with a DIFFERENT task in the same worktree = never: they share the working copy, git state and dev ports; a different task gets its own worktree.
2. `orca terminal list --worktree id:<worktreeId> --json` → handle
3. `orca terminal wait --terminal <handle> --for tui-idle --timeout-ms 60000 --json`
4. `orca orchestration task-create --spec "..." --json` → task_id
5. `orca orchestration dispatch --task <task_id> --to <handle> --inject --json`
6. Coordinator runs a rolling LOOP of `check --wait --types worker_done,escalation,decision_gate --timeout-ms 900000` - one message per call, N workers ≥ N iterations. A timeout = liveness checkpoint, not failure - and the idle-scan point (see § Idle backlog scan).

Before any (re)dispatch: `dispatch-show --task <id>` - never double-dispatch a live handle.
After `worker_done`: the coordinator MUST close the task itself - `task-update --id <task_id> --status completed --json` (verified live: status does NOT flip automatically, tasks dangle as "dispatched").
Disposable worker cleanup = `worktree rm --worktree <sel> --force` directly (destructive - user yes first), NOT `terminal close`: closing the last pane auto-spawns a fresh empty default-agent session and the workspace lingers. Orca preserves the branch on rm - delete it separately if the commits are junk.

Untracked path (`--prompt` or `terminal send`) only for lightweight/interactive sessions (brainstorm, infra work): then the prompt MUST embed a comms protocol (e.g. file bus `/tmp/<name>-bus.md`, STATUS/QUESTION/BLOCKED lines) - the channel is decided BEFORE launch, no retrofit.

**Worker prompt must always contain:** task spec + acceptance criteria; `interaction:` mode; comms protocol; reporting block (what goes to the bus, what to the tracker); branch rules (own branch, PR at end, never push main); required skills to invoke (see the role registry below); forbidden files (zones owned by parallel workers); **role line: "you are a WORKER, not an orchestrator: no orca worktree/terminal create, no orchestrator skills; parallelism = subagents inside your session; need siblings = ask the coordinator"**. Workers don't read your global config or KB rules - the prompt is their only law.

**Role-skill registry (embed in the prompt as "invoke skill X as your first action"):** executor worker (code/QA/infra, multi-phase plan) → `worker-mode`; PR reviewer → `pr-review` (one skill, code+security passes, read-only). Both ship in this repo's `skills/` directory. A role's whole policy lives in its skill - do NOT duplicate it into the prompt (prompt = task + protocol + zones). There are no other role skills; frontend/backend specialization = task content, not a role.

## Reading workers

- FORBIDDEN to rely on `terminal read` / screenshots of agent TUIs for results (agent CLIs draw in the alternate screen buffer - you get a spinner). Results come ONLY via bus messages + artifacts (git diff/log in the worktree, files, tracker comments).
- `terminal wait --for tui-idle` = heuristic with false positives, not a "done" signal.
- Liveness: bus silent >10 min with idle signals → neutral ping via `terminal send` ("if you are waiting for a confirmation - write to the bus what exactly"). Coding tasks legitimately stay silent 15-60 min - don't kill them.
- Dead worker: 2 empty wait windows → `worktree ps` (⚠️ main checkouts are invisible there - also run `terminal list`) → terminal gone = task → Blocked, inspect the worktree diff for salvage, respawn only via user.

## Models & budget

Routing FRAMEWORK - bind to your subscriptions:

| Work | Run on | Effort |
|---|---|---|
| Orchestrator | [your frontier model] | normal; high for wave planning |
| Code: hard architecture / security / nasty debug | [frontier] | high/max on spec + debug |
| Code: features, tests, refactors (DEFAULT) | [strong coding tier, e.g. Opus-class] | normal |
| PR review | [strong tier], fresh session, SEQUENTIAL queue | high |
| Docs/copy review, summaries | [mid tier, Sonnet-class] | normal |
| Mechanical batch edits | [fast tier, Haiku-class] | low |
| Web research / extraction | cheap non-frontier models via a router (don't burn frontier quota on reading pages) | - |
| DevOps/terminal grind, long autonomous runs, second opinion | second pool if you have one (e.g. Codex CLI on a ChatGPT plan) | medium |

- Budget guard: a wave of 3+ frontier workers → tell the user the cost in words BEFORE launch.
- Second-pool quotas are usually tight (e.g. ChatGPT Plus ≈ 20-100 msgs/5h): plan on the LOW bound, count tasks before dispatch; quota dies mid-task → worker checkpoints to the bus → reroute to your main pool in the SAME worktree, tell the user. Never silently wait out the quota window.
- Code is never written by the fast/mid tiers to save money - save on mechanics and research, not on code.

## Permissions (workers)

- `claude --permission-mode acceptEdits` + a curated allowlist in the repo's `.claude/settings.json` (allow: test/build/lint runners, git add/commit/push + `--force-with-lease` to OWN branch, project CLIs; deny: push to main, force-push elsewhere, rm -rf outside the worktree, prod deploys, deleting external resources). First fleet in a new repo = verify/create the allowlist.
- Full bypass mode: only short mechanical work, isolated worktree, no secrets, flagged to the user.
- Codex workers: set sandbox/approval flags explicitly (`codex exec --sandbox ...`) or they hang invisibly on approvals - and you cannot see their TUI.

## Conflicts & tracker

Pre-flight before taking a task: read the issue + its `## Exec` block if present (`env / agent / interaction / exclusive / depends (hard|soft)` - a hint, re-verify) → `worktree ps` + `terminal list` (who's alive) → exclusive resources → file zones vs running workers → hard depends. Verdict to the user in one line.
- Exclusive resources (the REAL desktop browser automated via MCP/CDP = ONE browser; emulator; prod DB): keep a lease table in the checkpoint; default = the second task queues until the holder's worker_done.
- **Browser routing (verified live):** default for worker QA/UI tasks = **Orca's built-in browser** (`orca goto/snapshot/click/eval`; device emulation exists: `orca exec --command 'set device "iPhone 14"'`). Its partition is persistent on disk and SHARED across all worktrees: log in once - the whole fleet has the session, it survives restarts; it needs NO exclusive lease (kills the one-browser queue). `tab profile create/clone` gives isolated login sessions when sharing is wrong. The real desktop browser (CDP / extension MCP) = only tasks needing the user's personal profile; its "allow" click is the tool's human gate - the orchestrator NEVER tries to self-approve it. Logins/registrations/passwords in ANY browser are typed by the USER only; the agent drives the already-authenticated session. State each worker's browser surface in its prompt.
- UI→backend→integration chains: 2 parallel workers + 1 joiner after both.
- Closing discipline: Done only with "what was done + links (PR/commits/KB)"; blocked = Blocked + reason; In Progress without a live session = fix it to Blocked. Create an issue when work outlives the session/day or has follow-ups; otherwise the KB daily note is enough.

## PR flow

Worker: branch → PR (template: what/why/tests/issue link) → never merges. Reviewer = separate fresh strong-tier session (sequential queue) running the `pr-review` skill: code + security passes over the full diff, verdict file + worker_done. Public repository: a verdict containing vulnerabilities is NOT posted to the PR before fixes land (the orchestrator posts after). Hygiene: no secrets/.env/dumps/temp/debug routes in the diff; .gitignore current. The orchestrator owns merge ORDER; after PR#1 merges, the rebase of PR#2 goes to its worker (`--force-with-lease` own branch); a non-trivial conflict = re-review. Merge gate: "PR #N: <gist>, review clean, merge?" → the USER merges (or says "merge"). Small PRs (no logic/schema/secrets, 1-2 files): inline review by the orchestrator, the user merge gate stays. Git destructive ops = explicit user yes.

## Interaction modes & escalation

Prompt flag `interaction: orchestrator-led` (default - the orchestrator answers worker questions from the KB/specs/recorded decisions; unknown = escalate) | `user-led` (brainstorm/design sessions - the user answers in the visible panel; relay = worker writes QUESTION to the bus / decision_gate, the orchestrator brings it to the user VERBATIM). Switchable mid-flight. Always escalate regardless of mode: money/logins/KYC; taste calls; irreversible actions.

## Idle backlog scan (between signals)

A long wait is not dead time: at wait checkpoints the orchestrator looks for what else can run in parallel.

- **Trigger:** a wait checkpoint - an empty `check --wait` window; a worker_done processed with nothing new to dispatch; a subagent finished and hands are free. NOT on a timer and NOT on every STATUS message (most STATUS updates change nothing about what can be parallelized).
- **Scope:** open tracker tasks of the CURRENT product's projects + the "ready next" list from the KB checkpoint. Don't scan other products: the user picks the day's focus.
- **Launch rights:** ready-next items (already user-approved) → take them yourself within the matrix (in-session/subagent; spawning a worker still goes through a proposal). New backlog candidates → ONE-line proposal to the user: task, env per the matrix, rough cost, conflict pre-flight verdict. No "yes" = no launch (the budget gate).
- **Anti-nag (ledger):** every shown candidate goes into the wave-checkpoint ledger: `candidate → verdict (launched/declined/deferred) → reason`. Declined = silence about that candidate. Re-offer ONLY on a state delta of that specific candidate (a blocker resolved, a worker slot / exclusive resource freed, the task edited in the tracker) - and the proposal must say WHAT changed; or when the user themselves asks "what can we run". A repeat with no new information is nagging.
- **New tasks:** propose creating a task only from a VERIFIED observation (a fact from the bus / a check / a loose end), never from "we could also..." brainstorming. Max 1-2 per proposal; created in the tracker only after the user's yes.
- **An empty scan** (no candidates / everything in the ledger with no delta) = silence, just the next wait window. Don't report "checked, nothing found".

## Reporting & context lifecycle

- Sources of truth, kept separate to avoid drift: **tracker = task event log** (workers + orchestrator write), **KB daily note = day narrative** (orchestrator only, written AS EVENTS HAPPEN), **KB project checkpoint = phase boundaries** (orchestrator only). Workers never write the KB directly (research subagents excepted if your KB rules allow).
- Wave checkpoint schema (write at phase boundaries AND before any compact): task_id↔handle↔worktree/branch↔issue-id; resource leases; open PRs + review status; next ready tasks; idle-proposal ledger (candidate → verdict → reason).
- Restart: read checkpoint → `worktree ps`+`terminal list` → drain the bus filtered by checkpointed task_ids → resume the loop. The bus is runtime-global and survives orchestrator restarts.
- Compact at ~70% context (proxies: context warnings, re-asking settled things) in a quiet moment, checkpoint first. Emergency at ~85%: checkpoint → announce a supervision gap → compact unconditionally. A user message always preempts the wait loop: quick matrix+conflict check → "join wave / queue / pause worker X".
- Keep the orchestrator thin: big files/diffs/logs go to subagents, summaries come back.

## Knowledge correction (duty, not option)

When a verified observation (own test, smoke run, fresh docs, a worker report) CONTRADICTS something recorded - in this skill, the KB, or the tracker - the orchestrator updates the source itself, immediately, without being asked:
- Skill wrong → fix the skill (the failed observation IS the failing test; log what changed).
- KB note wrong → patch it, bump its updated date.
- Tracker task stale → fix status + comment why.
- Always state the correction to the user in one line ("updated X: was A, turned out B").
Never leave knowledge you know to be false in any store; never fix only one store when several repeat the claim (grep for the claim across KB + skills before closing).

## Skill lifecycle (self-improvement loop, human-gated)

Borrowed from GenericAgent/OpenSkill-style self-skill research with their documented failure modes designed out (grow-only skill dumps, self-validating loops):
1. **Detect**: the same multi-step pattern done manually a 3rd time; a repeated mistake; a worker explicitly missing a capability.
2. **Propose, never auto-create**: one line to the user - "I see pattern X, propose skill/rule Y". The human gate is mandatory.
3. **Bloat check before authoring**: search existing skills + KB for overlap; prefer UPDATING an existing skill (upsert applies to skills too).
4. **Author via TDD for skills** (RED baseline = the actual observed failure; GREEN = replay the same scenario with the skill present).
5. **Anchor every mechanic claim** to something verifiable (command --help, a smoke run, a doc URL) - never to memory.
6. **Log every skill change** (changelog line; the change states what observation forced it).

## Red flags - STOP, you're about to violate

- Reading a worker's TUI to "see how it's doing" → bus + artifacts only.
- Spawning with `--prompt` for a real coding task "to save time" → tracked dispatch exists for a reason.
- "I'll do it on main, it's quick" while a worker runs on the repo → worktree.
- Launching codex without sandbox/approval flags → invisible hang.
- A 5-worker wave without telling the user the cost → budget guard first.
- Inventing a flag/model name from memory → verify with --help/docs.
- Closing the day with tracker tasks In Progress and no live sessions → fix statuses.

| Excuse | Reality |
|---|---|
| "terminal read will show the result" | Alternate screen = spinner. Results = bus + git diff. |
| "I'll retrofit comms later" | The preamble only injects at dispatch. Channel BEFORE launch. |
| "The CSS fix is atomic, main is fine" | Atomicity doesn't cancel the parallel worker on the repo. Worktree. |
| "Codex is cheaper, route everything there" | Tight quota. Count before dispatch, route by task type. |
| "Skill invocation slows me down, task is simple" | A single read-only subagent is exempt. Everything else - one-line matrix verdict. |
| "I'm idle - I'll just take a backlog task" | Self-serve = ready-next from the checkpoint only. New backlog items = one-line proposal, wait for yes. |
| "I'll propose it again, maybe they changed their mind" | A decline in the ledger = silence until that candidate's state changes. Repeating without new information is nagging. |
