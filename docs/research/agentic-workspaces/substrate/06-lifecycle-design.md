---
title: "Lifecycle design"
status: research
scope: substrate
---

# Lifecycle Design

Explicit lifecycle extension points were the strongest single idea to come out of
surveying existing worktree managers. The useful part is the lifecycle
abstraction itself — not any particular tool's opinionated merge workflow bolted
on top of it.

## Proposed minimal lifecycle

```text
create
  ├── pre-create      blocking
  ├── create worktree
  ├── post-create     blocking setup
  └── post-create-bg  background processes

enter
  ├── pre-enter
  ├── navigation/runtime focus
  └── post-enter

archive
  ├── pre-archive
  ├── stop/detach runtime
  ├── preserve workspace
  └── post-archive

reopen
  ├── pre-reopen
  ├── restore worktrees from metadata
  ├── post-create   only for setup invalidated while archived
  ├── post-create-bg
  └── post-reopen

remove
  ├── safety analysis
  ├── pre-remove
  ├── detach worktrees
  ├── move disposable data to trash
  └── post-remove

gc
  └── delete eligible trash/stale state
```

Reopen deliberately reuses `post-create` rather than inventing a parallel
`post-reopen-setup` event: the question it answers is the same one creation
answers — is this worktree ready to work in — and a hook author should not have
to write it twice.

What differs is that reopen must decide *which* setup is still valid. Archiving
detaches worktrees but does not freeze the world: lockfiles move, toolchains
change, and anything left on disk may have been garbage-collected. So reopen
should treat recorded bootstrap state as a claim to re-validate rather than a
fact, and re-run what no longer holds. Being able to say "already satisfied,
skipping" is what makes reopen cheaper than create — and it depends entirely on
the idempotency described below.

## Blocking versus background hooks

Blocking hooks establish invariants.

Examples:

```text
uv sync
pnpm install
generate config
database migration
```

A failed blocking hook means the workspace is not fully ready.

Background hooks start disposable runtime activity.

Examples:

```text
dev server
file watcher
local worker
```

These should be tracked and cancellable rather than making creation wait forever.

## Hook trust

Checked-in hook definitions are executable code.

Project-level hooks should require explicit approval before first execution and again when their executable definition changes.

Local user configuration can have a different trust policy.

## Idempotency

Bootstrap should be safe to retry.

A workspace should be able to represent:

```text
created
bootstrap-failed
ready
archived
remove-pending
```

rather than pretending creation is atomic when five external commands are involved.

## Trash before delete

Removal can first move disposable workspace data to a trash location and detach Git state safely.

Permanent deletion can happen asynchronously or during GC.

This improves perceived performance and creates an undo/recovery window.
