---
title: "Tool landscape"
status: research
scope: references
---

# Tool Landscape

Notes on tools this repository does not own, kept as design references.

**These are dated third-party observations, not authoritative.** They record how
each tool looked during the survey that produced these notes. Nobody who
maintains these tools has reviewed these pages, and any of them may have moved
on. Do not cite them as current fact; re-check before acting on one. gion's own
assessment lives in [gion/](../gion/) and is the only part of this research
grounded in code that can be verified from this repository.

## Categories

The tools fall into different architectural categories.

| Tool | Primary abstraction | Strength |
|---|---|---|
| gion | declarative multi-repo workspace | task composition, shared stores, plan/apply |
| Worktrunk | repository worktree | excellent lifecycle and UX |
| GTR | repository worktree | pragmatic complete feature set |
| gwq | global worktree inventory | navigation/discovery |
| git-wt variants | thin Git wrapper | minimal native ergonomics |
| Mani | repository workspace | long-lived repo inventory/bootstrap |
| Herdr | terminal workspace | processes, panes, agent sessions |

The mistake would be comparing every row as if they solve the same problem.

## Emerging preference

Use a task-first manager for workspace composition and borrow Worktrunk-quality lifecycle primitives.

Keep Worktrunk/GTR as references for ergonomics and safety rather than automatically adopting their entire workflow model.

## Pages

- [Worktrunk](worktrunk.md) — repository-scoped lifecycle and UX reference
- [GTR and thin managers](gtr-and-thin-managers.md) — GTR, gwq, `git-wt` variants
- [Herdr](herdr.md) — terminal runtime and the runtime-adapter argument
