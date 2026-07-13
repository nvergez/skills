---
name: orchestrate-tickets
description: Orchestrate parallel implementation of the tickets produced by /to-tickets. Builds the blocked-by dependency graph, spawns one Orca child worktree + agent per unblocked ticket, loops implementation and /code-review in each worker until no findings remain, merges each child branch back into the integration branch, and repeats until every ticket is done.
disable-model-invocation: true
metadata:
  author: nvergez
---

# Orchestrate Tickets

Implement a full set of tickets produced by `/to-tickets` (mattpocock/skills), in parallel, using Orca (stablyai/orca) for isolation and coordination.

You are the **coordinator**. You stay in the current worktree, on the **integration branch** (the feature branch the tickets belong to). Workers run in Orca child worktrees, one per ticket. You never implement tickets yourself — you build the graph, dispatch, supervise, merge, and repeat.

The core correctness rule: **a child worktree is created only when its ticket becomes unblocked**, so it branches off the integration branch *after* all its blockers have been merged. Never pre-create worktrees for blocked tickets.

## Arguments

`/orchestrate-tickets [tickets-dir] [max-parallel] [worker-agent]`

- `tickets-dir` — directory of ticket files, e.g. `.scratch/<feature-slug>/issues/`. If omitted and exactly one `.scratch/*/issues/` directory exists, use it; otherwise ask.
- `max-parallel` — max concurrent workers. Default: 3.
- `worker-agent` — Orca agent id for workers (`claude`, `codex`, ...). Default: `claude`.

## Preconditions — verify all before starting, fail fast

1. **Orca is running**: `orca status --json` succeeds (on Linux outside Orca terminals the binary is `orca-ide`; bare `orca` is the GNOME screen reader). Orchestration is enabled: `orca orchestration task-list --json` succeeds (if not, the user must enable it in Settings > Experimental).
2. **You are on the integration branch** with a clean working tree. If the current branch is the repo default branch (`main`/`master`), stop and ask the user to create or name a feature branch first — children merge back into this branch.
3. **Tickets parse**: every file in `tickets-dir` has a `**Status:**` line and a `**Blocked by:**` line (the `/to-tickets` local template).
4. **Workers can reach the skills they need**: `tdd` and `code-review` from mattpocock/skills must be available in a *fresh* worktree — either committed in the repo (`.claude/skills/`) or installed globally (`npx skills add mattpocock/skills -g`). If neither, stop and tell the user.

## Ticket status convention

This skill reads and writes the `**Status:**` line of each ticket file (in **your** worktree — it is the durable source of truth):

- `ready-for-agent` — not started (as written by `/to-tickets`)
- `in-progress` — dispatched to a worker
- `done` — merged into the integration branch
- `failed` — gave up after retry; its dependents stay blocked

A ticket is **unblocked** when every ticket on its `Blocked by` line is `done`. The **frontier** is all unblocked tickets not yet `in-progress`/`done`/`failed`.

**Resume:** if invoked again mid-flight, tickets marked `in-progress` with a live child worktree (`orca worktree ps --json` + `orca orchestration task-list --json`) are re-attached and supervised; `in-progress` with no live worktree is reset to `ready-for-agent` and re-dispatched fresh.

## Process

### 1. Build the dependency graph

Parse every ticket's number, title, and `Blocked by` edges. Validate: every referenced blocker exists, and the graph has no cycles — on either error, stop and report; the user fixes the tickets first.

Print the plan as waves (wave 1 = no blockers, wave 2 = blocked only by wave 1, ...) with the parallelism limit, then start. Do not wait for approval — `/to-tickets` already had the user validate the edges.

### 2. Start workers for the frontier

For each frontier ticket (up to `max-parallel` concurrent workers):

```bash
orca worktree create --name <NN>-<slug> --parent-worktree active \
  --base-branch <integration-branch> --agent <worker-agent> --json
orca terminal wait --terminal <handle> --for tui-idle --timeout-ms 120000 --json
orca orchestration task-create --spec "<worker brief — template below>" --json
orca orchestration dispatch --task <task_id> --to <handle> --inject --json
orca worktree set --worktree id:<worktree-id> --workspace-status in-progress --json
```

- `--base-branch <integration-branch>` is deliberate: this is stacked work; children must branch from the integration branch including previously merged tickets.
- From the create JSON, record the mapping **ticket → worktree id, branch name, agent terminal handle** (`result.agentTerminalHandle`, falling back to `result.startupTerminal.handle`). If a handle later goes stale, re-resolve via `orca terminal list --worktree id:<worktree-id> --json` and continue with the replacement only — never dual-send.
- Set the ticket's `**Status:**` line to `in-progress`.

### Worker brief template

The spec passed to `task-create` must inline the **full ticket file content** — `.scratch/` is often gitignored, so the ticket file may not exist in the child worktree. Note: Matt's `/implement` skill is user-invoked-only (`disable-model-invocation`), so a worker cannot load it from an injected prompt; its process is embedded below instead, and the worker uses `/tdd` and `/code-review` (both model-invocable) directly.

```text
You are implementing one ticket of the feature '<feature-slug>' in an isolated
Orca worktree. Work only on the current branch. Commit here; never push, never
merge, never switch branches. Your coordinator merges your branch when you are
done. The integration branch is '<integration-branch>'; your review fixed
point is: $(git merge-base HEAD <integration-branch>).

Step 1 — Implement the ticket below. Use the /tdd skill where possible, at
sensible seams. Run typechecking regularly, single test files regularly, and
the full test suite once at the end. The ticket's acceptance criteria define
done.

Step 2 — Review loop: run the /code-review skill with the fixed point above.
The ticket below is the spec for its Spec axis. Fix every finding on both
axes, commit, and re-run /code-review. Repeat until a round reports no
findings. Hard cap: 5 rounds — if findings persist, stop and report them.

Step 3 — Verify every acceptance criterion and that the full test suite
passes. Commit all work.

Then send worker_done exactly once (exact commands are in your preamble),
with filesModified and a 3-sentence summary. If you are blocked, use ask; a
failure is still a worker_done with subject "Failed: <reason>". Never use
AskUserQuestion.

=== TICKET <NN> — <title> ===
<full ticket file content, verbatim>
```

### 3. Supervise

Run a rolling wait loop — never sleep/poll, never idle-chat:

```bash
orca orchestration check --wait --types worker_done,escalation,decision_gate --timeout-ms 900000 --json
```

- Each return yields **one** message; loop again immediately after handling it.
- A timeout or `{count:0}` is a checkpoint, not a failure — long tickets run 15–60 min. Check liveness (`orca orchestration task-list --json`, `orca terminal read`, `orca terminal wait --for tui-idle`) and keep waiting. Never stop or restart a worker for mere silence.
- `decision_gate` / `ask`: answer from the tickets and feature context if the answer is determinable; otherwise ask the user, then `orca orchestration reply --id <msg_id> --body <answer> --json` and keep waiting.
- `escalation`: resolve if you can, else surface to the user.
- A valid `worker_done` auto-completes the task — do not follow with `task-update`.

### 4. Land a finished ticket

On a successful `worker_done`, in **your** worktree on the integration branch:

```bash
git merge --no-ff <child-branch> -m "ticket <NN>: <title>"
```

- On conflicts, use the `/resolving-merge-conflicts` skill (mattpocock/skills) — resolve hunk by hunk, never abort.
- After merging, run the project's quick verification (typecheck + the tests touching the merged files). If it fails because of the merge, fix forward on the integration branch.
- Set the ticket's `**Status:**` line to `done`, then:

```bash
orca worktree set --worktree id:<worktree-id> --workspace-status completed --comment "merged into <integration-branch>" --json
```

Merge completions one at a time, in arrival order — never merge two child branches concurrently.

### 5. Refill the frontier

After each landing, recompute the frontier — landing a ticket may unblock others, and their fresh worktrees will now include the merged work. Start new workers (step 2) up to `max-parallel`. Loop steps 3–5 until no ticket is `ready-for-agent` or `in-progress`.

### 6. Finish

1. Run the **full test suite** on the integration branch.
2. Run `/code-review` once across the whole feature: fixed point = `$(git merge-base HEAD <default-branch>)`. Fix findings directly on the integration branch.
3. Remove merged child worktrees: `orca worktree rm --worktree id:<worktree-id> --force --json` (only ones whose ticket is `done` — keep `failed` ones for inspection).
4. Report to the user: tickets done, tickets failed and why, integration branch state, and suite/review results.

## Failure handling

- **Failed worker_done** (`"Failed: <reason>"`): retry once — remove that child worktree, create a fresh one (it re-branches off the current integration branch), and include the failure summary in the new brief. If the retry also fails, set the ticket to `failed`, leave its dependents blocked, continue with independent tickets, and report at the end.
- **Dead worker** (terminal gone, no worker_done): treat as a failed attempt and apply the same retry rule.
- Orca circuit-breaks a task after 3 consecutive dispatch failures — if that happens, create a *new* task for the retry rather than re-dispatching the broken one.
