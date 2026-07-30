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

Checked-in hook definitions are executable code. Project-level hooks should
require explicit approval before first execution, and again when their
definition changes. Locally-configured hooks, which the user wrote themselves,
can have a different policy.

That much is easy to state and easy to get wrong, so the mechanics matter.

**What approval is bound to.** An approval should record a digest of the
resolved hook definition, scoped to the repository and the hook event. Comparing
digests on each run is what makes "again when their definition changes"
enforceable rather than aspirational.

**The limit of that guarantee, stated plainly.** A digest over the hook
definition does not cover what the definition *invokes*. A hook declared as
`./scripts/worktree-bootstrap` has a stable definition while the script it runs
changes freely with every pull — so approving the hook once approves an
indirection, not a payload. Options are to digest the referenced scripts too
(brittle: they have their own dependencies), to treat any change to the
checked-out tree as invalidating (safe, unusably noisy), or to accept the
limitation and be honest that hook approval gates *declaration* rather than
behavior. There is no clean answer; a design that does not pick one has silently
picked the third.

**Where approvals live.** Outside the repository. An approval stored in the tree
it authorizes is self-certifying — a pull could deliver both a new hook and its
own approval.

**Failure handling.** Unapproved, changed-since-approval, and declined all
resolve the same way: the hook does not run, and the workspace stops in
`bootstrap-failed` with the reason recorded. Never silently skip a blocking hook
and continue — a workspace that reports ready without its invariants established
is worse than one that reports failure.

**Non-interactive contexts.** This is where trust models usually break. An agent
or CI run has nobody to prompt, and the tempting resolutions — auto-approve when
there is no TTY, or let `--no-prompt` imply consent — both convert the trust gate
into an announcement. Unattended runs must fail closed on an unapproved hook.
Pre-approving out of band (an explicit, auditable approval step before the
unattended run) is the escape hatch, not inferring consent from the absence of a
human.

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
