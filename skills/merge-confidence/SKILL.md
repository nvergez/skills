---
name: merge-confidence
description: Grade a PR or branch across five evidence axes and derive a merge-confidence verdict — band + index computed mechanically from the grades, every axis backed by cited evidence.
disable-model-invocation: true
metadata:
  author: nvergez
---

# Merge Confidence

Produce a **verdict** on whether a branch or PR is safe to merge: a band (**green** / **amber** / **red**) and an index (0–100), derived mechanically from five axis grades. You never pick the number — it falls out of the grades. The verdict is the deliverable; the merge decision stays with the user.

## Arguments

`/merge-confidence [target]`

- `target` — a PR number (`123`, `#123`) or a branch name. Default: the current branch.

## Preconditions

1. The target's head is checked out locally (for a PR: `gh pr checkout <n>`, or a worktree) and the working tree is clean — evidence must reflect the branch, not local edits.
2. `git merge-base HEAD <base>` resolves, where **base** is the PR's base branch or the repo default branch. This merge-base is the **fixed point**; every axis grades the diff from the fixed point to the head.

Also locate the **spec** if one exists: the PR description, its linked issue, or the ticket file the branch implements (`.scratch/*/issues/`).

## Grading

Each axis gets exactly one grade — the first line that matches, top to bottom, within its axis below. Grades are decided only by the stated observables, never by overall impression.

**`unknown`, never skipped**: an axis whose evidence you could not obtain is graded `unknown` with the reason cited (no CI configured, no spec found, review skill unavailable, suite won't run).

### 1. Correctness

Run the `/code-review` skill (mattpocock/skills) with the fixed point above; if a spec exists, it is the input to the review's Spec axis.

- `fail` — the review reports any correctness finding
- `warn` — only quality findings (reuse, simplification, efficiency)
- `pass` — a clean round
- `unknown` — `/code-review` unavailable in this environment

### 2. CI / checks

Checks on the head SHA (`gh pr checks`, or the branch's latest run).

- `fail` — a required check failing
- `warn` — checks pending, stale (not on the head SHA), or only non-required ones failing
- `pass` — all checks green on the head SHA
- `unknown` — no PR and no CI configured

### 3. Tests

Run the project's suite locally. "Exercised" means a coverage run shows the hunk executed, or you traced a specific test to it — name which.

- `fail` — suite red, or no test exercises any changed source hunk
- `warn` — suite green but some changed source hunks are unexercised
- `pass` — suite green and every changed source hunk is exercised
- `unknown` — suite won't run locally and there is no CI signal to fall back on

### 4. Spec

Map each acceptance criterion to the diff hunks implementing it, and each source hunk back to a criterion.

- `fail` — a criterion with no implementing change
- `warn` — every criterion covered, but the diff carries changes no criterion asked for, or criteria too vague to map
- `pass` — every criterion maps to hunks and every source hunk serves a criterion
- `unknown` — no spec found

### 5. Blast radius

Classify what the diff touches: leaf modules vs shared modules (imported widely), and sensitive surface (auth, payments, data migrations, public API, config/schema).

- `fail` — sensitive surface changed with no test in the diff exercising that change
- `warn` — shared modules or sensitive surface touched (with tests)
- `pass` — changes confined to leaf modules and their tests

## Verdict

Both are computed, in this order:

- **Band**: `red` if any axis is `fail`; else `amber` if any axis is `warn` or `unknown`; else `green`.
- **Index**: sum the axes — `pass` = 20, `warn` = 10, `unknown` = 5, `fail` = 0.

The band is the verdict; the index ranks work within a band (a red 80 and a red 40 are both no-go, one is closer).

## Report

Lead with the verdict line, then the evidence table, then the single next action:

```text
Verdict: AMBER — 75/100

| Axis         | Grade   | Evidence                                        |
| Correctness  | pass    | /code-review clean, fixed point abc1234         |
| CI           | pass    | 4/4 checks green on def5678                     |
| Tests        | warn    | suite green; src/billing/rate.ts:40-62 unexercised |
| Spec         | pass    | 3/3 criteria mapped (ticket 04)                 |
| Blast radius | warn    | touches shared module src/lib/http.ts           |

Next action: add a test over rate.ts:40-62 — flips Tests to pass, index to 85.
```

- Every evidence cell cites an observable: a finding, a check name + SHA, a file:line range, a criterion number. A cell that would read like an impression means the axis is really `unknown`.
- **Next action** is the cheapest change that flips the worst-graded axis.

The run is complete when all five axes carry a grade and a citation, and the report has been delivered — reported, not acted on: no merge, no fixes.
