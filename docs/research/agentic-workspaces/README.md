---
title: "Agentic workspaces & Git worktrees (research)"
status: research
---

# Agentic Workspaces & Git Worktrees

Research notes on building a scalable agentic engineering workstation around Git
worktrees, and where gion sits inside that architecture.

The central claim is that **a worktree is an implementation primitive; a task
workspace is the unit of work**. A task may involve one worktree from a
monorepo, several worktrees from a polyrepo system, task-local plans and
artifacts, one or more coding agents, dev servers and tests, and lifecycle setup
and teardown.

These are research notes, not specifications. `docs/spec/` remains the source of
truth for gion's behavior. Nothing here is committed roadmap.

## Separation of concerns

This material was originally written as a single wiki that was kept in more than
one worktree-manager repository at once. That does not survive contact with two
repos: the tool-neutral reasoning drifts, and each copy accumulates claims about
the other tool that its own maintainers never review.

It is therefore split into three buckets with different ownership rules:

| Directory | Scope | Rule |
|---|---|---|
| [`substrate/`](substrate/) | Tool-neutral architecture and requirements | No tool advocacy. Portable verbatim to any worktree-manager repo. |
| [`gion/`](gion/) | How gion maps onto the substrate | Only valid in this repo. Must cite gion code or specs. |
| [`references/`](references/) | Other tools evaluated as design references | Observations about tools we do not own. Dated, not authoritative. |

The test for which bucket a page belongs in:

- if it would read identically in a repo that is not gion → `substrate/`
- if it names a gion file, command, or spec → `gion/`
- if it describes a tool this repo does not ship → `references/`

`substrate/` is the shared half. When it changes, the change should be
reviewable by someone who has never read gion's source. `gion/` is the local
half and is expected to diverge from any sibling copy — that divergence is the
point, not drift.

## Contents

### Substrate — tool-neutral

- [Problem framing](substrate/01-problem-framing.md) — why worktrees became load-bearing, and the infrastructure/policy trap
- [Core insights](substrate/02-core-insights.md) — the task as unit of work, and the boundaries that follow
- [Requirements](substrate/03-requirements.md) — mandatory, desired, and explicitly out of scope
- [Architecture overview](substrate/04-architecture-overview.md) — layers and ownership boundaries
- [Monorepo and polyrepo](substrate/05-monorepo-polyrepo.md) — one model for both
- [Lifecycle design](substrate/06-lifecycle-design.md) — hooks, trust, idempotency, trash-before-delete
- [Storage and performance](substrate/07-storage-performance.md) — shared object stores and the costs that are not Git
- [Task workspace workflow](substrate/08-task-workspace-workflow.md) — create, enter, status, archive, reopen, purge

### gion — local

- [Positioning](gion/positioning.md) — what gion already is in this architecture
- [Gap analysis](gion/gap-analysis.md) — every substrate requirement scored against the current codebase
- [Adoption questions](gion/adoption-questions.md) — open decisions that block or shape adoption

### References — external tools

- [Tool landscape](references/README.md) — the categories these tools actually fall into
- [Worktrunk](references/worktrunk.md)
- [GTR and thin managers](references/gtr-and-thin-managers.md)
- [Herdr](references/herdr.md)

### Provenance

- [Exploration log](exploration-log.md) — how these conclusions were reached, in order

## Design principle

Keep the bottom layers boring.

Git should own Git. The workspace manager should own workspaces. The terminal
runtime should own terminal and process sessions. Specification and agent
systems should own tasks, plans, and execution policy.

The temptation to make one CLI own all of these is exactly how a useful worktree
wrapper slowly becomes a small civilization.
