---
title: "Worktrunk"
status: research
scope: references
---

# Worktrunk

Repository: `max-sixty/worktrunk`

## What it gets right

Worktrunk is one of the strongest references for repository-scoped worktree lifecycle UX.

Its valuable ideas include:

- branch-oriented worktree addressing,
- configurable paths,
- rich lifecycle hooks,
- blocking pre-hooks,
- background post-hooks,
- hook approval,
- safe removal,
- integrated-branch checks,
- trash/background deletion,
- shell integration,
- aliases/custom subcommands,
- interactive navigation/status.

## The lifecycle lesson

The best part to borrow is not `wt merge`.

It is the notion that worktree creation/removal is a lifecycle with explicit extension points and safety boundaries.

## Where its abstraction stops

Worktrunk is fundamentally repository-first:

```text
repository
├── branch A → worktree
├── branch B → worktree
└── branch C → worktree
```

It does not naturally make one task the parent of worktrees from several repositories.

That is acceptable for a monorepo, but less expressive for coordinated polyrepo tasks.

## Features that should stay optional/outside the substrate

- LLM-generated commit messages,
- model/API configuration,
- opinionated commit/merge pipeline,
- PR workflow ownership,
- AI-generated summaries.

These are useful features for some users, but they should not become dependencies of the workstation's foundational workspace layer.

## Role in this project

Treat Worktrunk as a design reference for:

```text
lifecycle
safety
navigation
extensibility
cleanup
```

rather than necessarily the final substrate.
