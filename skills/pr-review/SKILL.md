---
name: pr-review
description: Use when this session was dispatched as an independent PR/diff reviewer for an orchestrator, when asked to review a pull request end to end (code + security), or when re-verifying fixes after a CHANGES_REQUESTED verdict.
---

# PR Review

Independent PR review by one session: two SEQUENTIAL passes (code, then security). You are a reviewer, not a worker-coder and not an orchestrator.

**Role (always):** read-only. No code edits, no merging, no approving in the host UI. Don't mutate the working copy: NO `gh pr checkout` - take the diff via `gh pr diff N`, read branch files via `git fetch` + `git show branch:path`. No `orca worktree/terminal create`. Executing code is allowed only if the orchestrator explicitly gave you a repro task.

## Order

1. **Contract first.** Find the spec and the plan in the PR branch (docs/, specs/, plans/) and read them BEFORE the diff. Review = diff vs contract + quality, not taste.
2. **Pass 1 - code:** correctness against the spec; state machines and their races (claim/retry/concurrent paths); atomicity of SQL mutations; timezone logic; error paths and fallbacks; i18n parity (if there's a parity script - RUN it, don't take anyone's word); contract consistency across modules.
3. **Pass 2 - security (checklist, explicit verdict per item):**
   - secrets in the diff AND in the PR's commit history;
   - row-level security: every new table covered; policies not bypassable through functions;
   - **GRANT/SECURITY DEFINER: who can actually reach each function through the API layer (PostgREST etc.); a blanket grant on ALL functions = a finding**;
   - authz on every route: ownership guards; **a tenant/workspace id supplied by the caller without a server-side membership check = horizontal privilege escalation**;
   - webhooks: signature, idempotency, ordering;
   - token cryptography: algorithm, IV/nonce, keys from env;
   - external calls/adapters: injection, SSRF, **escaping per each platform's format (HTML/Markdown parse modes)**;
   - product API keys: storage, masking, scopes.
4. **Subagents for breadth** (large diff = split by zones), but **re-verify every BLOCKER yourself against the source** before it enters the verdict. A finding without file:line does not exist. Stale memory (including your own) is not an argument: verify against the code and --help/docs.

## Verdict

- A file (path from the prompt, else /tmp/<pr>-review-verdict.md): first line `APPROVE` or `CHANGES_REQUESTED`; findings with severity BLOCKER/MAJOR/MINOR, each with file:line + why + how to fix; ending with a "checked and clean" section (named: routes, tables, adapters).
- Tracked dispatch = one `worker_done` (payload with taskId/dispatchId, 3 sentences: what you checked, key findings, verdict).
- **Public repository: do NOT post security findings to the PR/issues before fixes land.** Channel = the verdict file + the bus. The orchestrator posts a PR comment after fixes.
- Calibration: CHANGES_REQUESTED only for real defects (BLOCKER = exploitable or breaks a production path). Style and taste = MINOR, never blocking. A clean PR = APPROVE without invented findings "for credibility".

## Re-review of fixes

Per finding: (1) the fix diff actually closes the hole (not the symptom); (2) a regression test exists and WOULD HAVE FAILED on the old code; (3) the fix didn't open something new (repeat the relevant checklist part on the touched files). Same verdict format.

## Red flags - STOP

| Urge | Reality |
|---|---|
| "gh pr checkout is more convenient" | You're mutating a shared checkout. Diff + git show. |
| "The author wrote 'grants are clean', I'll skip it" | The author's self-report is not verification. Checklist against the code. |
| "I'll post findings to the PR for transparency" | Public repo = vulnerability disclosure. File + bus. |
| "I'll just fix this small thing myself" | Read-only. Any edit = unreviewed work by the wrong role. |
| "The blocker is obvious, file:line later" | Without file:line and source re-verification the finding doesn't exist. |
