---
title: "Storage and performance"
status: research
scope: substrate
---

# Storage and Performance

## Git worktree storage

Worktrees share the repository's Git object database.

Conceptually:

```text
shared Git object database
├── main worktree
├── task-A worktree
├── task-B worktree
└── task-C worktree
```

This is much cheaper than cloning the repository for every agent.

A manager that maintains an explicit shared bare-repository pool makes this model
especially natural for task workspaces spanning several repositories: each
repository is cloned into the pool once, and every task attaches worktrees from
it rather than re-cloning.

Cloning once is not the same as fetching once. The pool is a cache of remote
state, so it goes stale, and a stale pool is worse than a slow one: a task cut
from a stale `origin/main` starts from the wrong base, and nothing about the
resulting worktree looks wrong. The pool must therefore be refreshed on a
defined schedule — at minimum before resolving the base ref for a new worktree,
and on explicit demand — rather than only when a repository first enters it.

Freshness is a correctness property here, not a performance one. Fetching is
cheap and bounded-parallelizable; silently branching from a stale base is
neither cheap nor visible.

## The expensive parts are usually elsewhere

Disk and startup costs commonly come from:

```text
node_modules/
.venv/
target/
build outputs
Docker images/volumes
databases
model artifacts
```

## Preferred strategy

Use shared immutable/content-addressed caches and independent mutable environments.

Examples:

- pnpm global content-addressable store,
- uv cache,
- Cargo registry/git cache,
- Go module cache,
- Nix store.

Then bootstrap each worktree from lockfiles.

This balances reproducibility with storage efficiency.

## Avoid casual mutable sharing

Directly symlinking mutable directories across worktrees can introduce cross-agent interference.

Be careful with:

- `.venv`,
- `node_modules`,
- build directories,
- SQLite databases,
- local generated state.

Copy-on-write/reflink strategies can be valuable where supported, but should be an optimization rather than a correctness dependency.

## Parallelism

The earlier custom Bash manager exposed a practical issue: serial operations across many repositories become slow.

A compiled Go manager can:

- fetch repositories concurrently,
- bootstrap independent repositories concurrently,
- aggregate errors,
- bound concurrency,
- cancel dependent work,
- stream per-repository status.

This is especially important for polyrepo task creation.
