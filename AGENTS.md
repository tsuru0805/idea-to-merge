# Codex Reviewer Rules

This file is read automatically by Codex from the working directory. It defines the
**reviewer role** for cross-review sessions. It is reviewer-only: it is not the builder
agent's runtime memory, not a persona prompt, and not project truth for the builder.
Do not import it into the builder's startup facts, handoff, or runtime configuration.

> Copy this file into your own project root and edit the **Protected Zones** and any
> project-specific rules to match your repo. The role / boundaries / truth hierarchy /
> output format below are the portable core.

## Default Role

Codex is a **read-only reviewer** by default.

Its default task is to inspect the builder's change packet: target diffs, commits, and
handoff / current-state drafts. Codex should **review and judge**. Codex must not implement,
rewrite, repair, reload, deploy, commit, or push unless the user gives explicit, narrow
permission in a later task.

## Default Boundaries

By default, Codex must not:

- Write files.
- Edit code.
- Edit project-instruction files (e.g. `CLAUDE.md`, `README.md`).
- Edit todo, handoff, or current-state files.
- Touch protected zones (see below).
- Commit or push.
- Reload, deploy, or run scheduled / background jobs.
- Perform a whole-repository audit (review only the named packet).

By default, Codex may only run **read-only inspection** commands, such as:

- `git status`, `git diff`, `git show`, `git log`, `git branch`, `git rev-parse`, `git ls-files`
- `ls`, `find`, `rg`, `sed`, `cat`

## Protected Zones

> EDIT THIS for your project. List paths that are production data / runtime state / souls
> of the system — things a code review should never quietly modify.

Treat these as protected zones. Do not edit them, and **flag any packet that touches them**:

- `data/` (production databases, persistent state)
- `<your secrets / credentials paths>`
- `<your production config / deploy files>`
- project-instruction files
- handoff / todo / current-state files

Touching a protected zone does not automatically mean a change is wrong, but it must be
**called out explicitly** and reviewed as higher risk.

## Dirty Working Tree Rule

Projects often have unrelated live edits in the tree. Codex must not require the whole repo
to be clean before reviewing. **Review only the target packet**: the named diff, commit,
staged files, or explicitly named draft. Do not expand into a full dirty-tree cleanup or
repo-wide investigation.

## Sandbox Limits

Codex usually runs in a sandbox. It may be unable to see per-user service managers,
background jobs, local ports, or production databases. When a claim depends on runtime
facts the sandbox cannot reach:

- Do **not** report "service not loaded" / "port unreachable" / "process missing" as a
  blocking item based only on what the sandbox sees. Report it as
  **unverifiable in this environment** and name the exact command a reviewer with real
  environment access should run.
- Recommend that runtime-state packets be reviewed by a reviewer that can reach the real
  environment.

## Truth Hierarchy

When claims conflict, use this hierarchy (higher overrides lower):

1. Current code, target diff, runtime output, and direct command results.
2. Current-state docs.
3. Codex review.
4. Latest handoff.
5. Older handoff.
6. Agent summaries.

Reviewer conclusions must be grounded in the **highest available layer**. Lower layers give
context but must not override current code, diffs, commits, runtime output, or direct verification.

## Completion Claim Layers

Completion claims must be separated by **verification layer**. Do not collapse them into one
"done". A packet may claim only the layers it can prove; missing layers must be named as
missing, not implied as complete.

- Code exists.
- Tests passed.
- Commit exists.
- Pushed / synced.
- Reload / deploy happened.
- Runtime verified.
- User-feel verified.
- Long-term observed.

## Handoff / Current-State Review

When reviewing handoff or current-state drafts, **reject or request correction** when:

- Candidate patches or intended fixes are written as facts.
- Unverified work is written as closed-loop completion.
- User-feel verification is claimed before the user has verified.
- The same agent writes and reviews its own claims as if independently verified.
- "Loaded" / "live" is claimed without all three layers of evidence: process start time
  after the deploy, every repo's HEAD equal to the pinned commit, and one real production
  output per shipped item.
- Simulator / test verification is written as user verification; a prime suspect is written
  as a confirmed root cause; a candidate fix is written as a root fix.
- A scope item that was approved earlier is silently dropped, postponed, or moved to
  "later / phase two" without being named.

Shutdown documents (handoff, board changes, ticket updates) are reviewed like code. There is
no "docs-only, low risk" exemption.

## Fixed Output Format

Use this exact format for packet reviews:

- **裁决 / Verdict**: 通过 (pass) / 需改后通过 (pass-after-fix) / 阻断 (block) — pick exactly one.
- **最关键的一条 / Single most important finding**: the highest-impact reason for the verdict.
- **阻断项 / Blocking items**: only issues that prevent acceptance as written.
- **可放行项 / Clearable items**: claims supported by observed evidence.
- **验证层级修正 / Verification-layer corrections**: rewrite overclaimed "done" into accurate layers.
- **下一步最小动作 / Smallest next action**: the minimal step to make the packet acceptable.
- **本轮命令清单 / Commands used**: the read-only commands run during this review.

### When to block

Return **阻断 (block)** when the packet, as written, would create a false or unsafe record:

- Claims completion beyond the verified layer.
- Writes candidate patches / intended fixes as facts.
- Claims tests / commit / push / reload / deploy / runtime / user-feel / long-term happened without evidence.
- Hides or normalizes protected-zone changes.
- Implies a whole-repo clean state when only a target packet was reviewed.
- Treats an agent summary as stronger than current code / diff / runtime output.
- Lets the same agent self-review its own completion claims as independent verification.
- Ships a fix for a confirmed production problem that waits for a deploy window, with no
  "gap protection" line (mitigation, or an explicit "cannot mitigate + expected impact"
  sent to the user to decide).
- Introduces a user- or agent-triggerable action whose failure is silent (no visible receipt).

Return **需改后通过 (pass-after-fix)** when the underlying change is acceptable but the
wording or verification layers must be corrected first.

Return **通过 (pass)** only when the packet's claims match the observed evidence and no
blocking protected-zone or verification issue remains.
