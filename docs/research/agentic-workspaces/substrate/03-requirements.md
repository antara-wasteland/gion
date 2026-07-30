---
title: "Requirements"
status: research
scope: substrate
---

# Requirements

## Mandatory

### Post-create bootstrap

A newly created worktree should be able to run deterministic setup such as:

```bash
uv sync
pnpm install --frozen-lockfile
git submodule update --init --recursive
direnv allow
./scripts/worktree-bootstrap
```

The hook should receive useful context such as:

```text
TASK_ID
WORKSPACE_PATH
REPO_NAME
WORKTREE_PATH
BRANCH
INDEX
```

**Trust is part of this requirement, not an enhancement of it.** Every command
above is defined by the repository being checked out — `./scripts/worktree-bootstrap`
plainly so, but `direnv allow` and package-manager lifecycle scripts equally, since
both execute code the checkout controls. Creating a worktree therefore means
executing untrusted content unless approval gates it.

So bootstrap must not run repository-controlled commands until they are approved,
and approval must be re-sought when the definition changes. Hook trust appears
again under *Strongly desired* below, where it covers the wider trust model
(scope, storage, and policy for locally-configured hooks); that placement is about
the machinery, and does not make approval optional for this requirement. A
manager that automatically runs checked-in setup on clone-and-create has built a
remote code execution path, not a convenience.

### Ergonomic CLI

Core operations should be cheap enough to become muscle memory:

```text
create
open / enter
list
status
exec
remove
cleanup
```

Interactive fuzzy navigation is useful, but every important operation should also be scriptable.

### Safe cleanup

Cleanup should distinguish:

- clean and integrated,
- dirty,
- unpushed,
- diverged,
- unknown,
- stale,
- archived.

Destructive operations should fail safely by default.

### Monorepo and polyrepo support

One task should be able to represent:

```text
TASK-A/
└── platform/          # monorepo
```

or:

```text
TASK-B/
├── frontend/
├── backend/
└── infra/             # polyrepo
```

without changing the overall workflow model.

### Efficient storage and execution

Avoid full clones per task.

Use shared Git object stores through worktrees/bare repositories and shared package caches where safe.

### Machine interface

Prefer:

- stable exit codes,
- JSON output,
- non-interactive flags,
- deterministic paths,
- explicit state.

Agents should not need to scrape pretty terminal output.

## Strongly desired

- hook trust/approval,
- blocking and background hooks,
- archive/reopen,
- trash-before-delete,
- doctor/repair,
- path templates,
- custom subcommands,
- shell navigation protocol,
- parallel operations across repositories.

## Explicitly not required at this layer

- LLM-generated commit messages,
- model API keys,
- task decomposition,
- autonomous merging,
- PR policy,
- CI decision policy,
- specification ownership.

Those belong above the workspace substrate.
