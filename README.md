# skills

Personal agent skills, installable with the [`skills` CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add nvergez/skills            # pick skills interactively
npx skills add nvergez/skills --list     # list what's here
```

The theme of this repo is **skills that orchestrate other skills** — encoding multi-skill workflows I used to re-explain by hand, mostly combining [mattpocock/skills](https://github.com/mattpocock/skills) (engineering flow: tickets, TDD, code review) with [Orca](https://github.com/stablyai/orca) (worktrees, agent orchestration).

## Skills

| Skill | What it does |
| --- | --- |
| [orchestrate-tickets](skills/orchestrate-tickets/SKILL.md) | After `/to-tickets`: build the blocked-by dependency graph, spawn one Orca child worktree + agent per unblocked ticket, loop implementation + `/code-review` in each until clean, merge each child branch back into the feature branch, repeat until all tickets are done. |
| [merge-confidence](skills/merge-confidence/SKILL.md) | Grade a PR or branch across five evidence axes (correctness via `/code-review`, CI, tests, spec, blast radius) and derive a merge-confidence verdict — band + 0–100 index computed mechanically from the grades. |

## Prerequisites

These skills drive skills from other repos, so those need to be installed where the agents run (globally is simplest, so fresh worktrees see them too):

```bash
npx skills add mattpocock/skills -g
npx skills add stablyai/orca -g
```

`orchestrate-tickets` additionally needs the Orca app running with the orchestration experimental feature enabled.
