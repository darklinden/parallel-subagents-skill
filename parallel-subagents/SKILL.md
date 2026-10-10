---
name: parallel-subagents
description: Dispatch multiple tasks to N parallel subagents with isolated worktrees under .subagents/, wave-based scheduling, and branch+push delivery. Use ONLY when the user explicitly requests it — e.g. types /parallel-subagents or clearly asks to "dispatch subagents in parallel" / "并行派发子 agent". Never trigger implicitly.
---

# Parallel Subagents

Dispatch a user-supplied task list to at most N parallel subagents. Each subagent works in its own clone under `.subagents/`, works on its own branch, and delivers by pushing that branch. Merge is always a human (or explicitly authorized main-agent) action — this skill never merges.

**N is a parallelism cap (to avoid freezing the machine), not the task count.** Wave size is `min(N, T)` where T is the number of tasks.

## Trigger and input

- Start ONLY on explicit user request (slash command or an unmistakable "dispatch subagents in parallel" instruction). Never auto-trigger.
- The trigger message must contain: a task list (one task per line) and N.
- If N is missing, ask once: "How many subagents should run in parallel?"
- If the task list is missing, ask once for it. **Task decomposition is NOT this skill's job** — never split a large goal into tasks yourself; only consume a list the user provides.

## Preflight checks (all mandatory, in order)

1. **git repo**: not a git repo → stop and ask the user what to do. No auto-fallback.
2. **remote**: no pushable remote → stop and ask the user.
3. **push access**: verify push permission (e.g. `git ls-remote origin` succeeds) → otherwise stop and ask.
4. **dirty main worktree**: uncommitted changes in the main worktree → stop and ask: commit first / stash / clone as-is. Default recommendation: commit first.
5. **.gitignore**: if `.gitignore` does not already ignore `.subagents/`, append it before doing anything else.

Do not proceed past a failed check without explicit user direction.

## Prepare work copies

- Reuse idle clones under `.subagents/` when available; clone fresh from the current project only if idle copies are short of the wave size.
- Exactly `min(N, T)` occupied directories during a wave — no extras.
- If idle copies exceed the needed count, use only what is needed; never delete or overwrite extras without user confirmation.
- Each clone excludes `.subagents/` (no recursive clone) and `.git/` of the source (avoid nested-repo accidents). Clone what is committed; dirty-worktree handling was already settled at preflight.

## Dispatch (wave execution)

- Split the task list into waves of at most N tasks.
- Within one message, launch the wave as N parallel Agent tool calls, each with `run_in_background: true`.
- Each subagent prompt MUST instruct:
  - `cd` into its assigned `.subagents/<dir>` as its working directory;
  - create a branch named `subagent/<task-slug>-<short-id>` from the **current checked-out branch HEAD** of the main repository (do not care about unrelated existing changes for branch base selection);
  - never touch the main branch;
  - when done, commit its own changes and **push its own branch only**;
  - never merge, never push to the main branch, never touch other `.subagents/` directories.

## Release criteria (per subagent)

Mark a subagent's directory released ONLY when all three hold:

1. Its Agent call has returned (process ended).
2. Its remote branch exists — verified by the main agent via `git ls-remote origin refs/heads/subagent/<task-slug>-<short-id>`.
3. Its working directory has no uncommitted changes.

If any condition fails: mark the directory **occupied/abnormal**, report to the user; do not silently release it.

## Wave gate

- The next wave starts as soon as **all released subagents of the current wave have pushed their branches** (gate = push, not merge).
- Merge does NOT block the next wave; the user interleaves merges at their own pace.

## Between waves

- For a released idle directory: `git fetch`, switch back to the main branch, and hard-sync to the latest — **only after** the release criteria are met.
- Reuse the directory in the next wave; do not re-clone when a valid idle copy exists.

## Delivery and merge

- Delivery = pushed branch. This skill never merges.
- Merge is performed by the human, or by the main agent only after the user explicitly authorizes it in a given turn.
- On merge conflict: stop and report. Do not force-resolve. The skill only guarantees branches are clean and mergeable, not conflict-free.

## Branch cleanup

- After the user confirms a branch is merged, do NOT delete it automatically. Ask once: "Branch <name> confirmed merged — delete local/remote copy?" Delete only on approval.

## Safety constraints

- Never delete, overwrite, or clean up any directory, change, or branch the user has not confirmed.
- Directory occupancy must be trackable and distinguishable (occupied / idle / abnormal) within the session.
- Never create more active work directories than the wave needs.
