---
title: "Task workspace workflow"
status: research
scope: substrate
---

# Proposed Task Workspace Workflow

## Create

```text
task-start TASK-123
```

Conceptually:

1. Resolve task/workspace definition.
2. Determine required repositories.
3. Resolve hook definitions and check approval. Any repository-controlled hook
   that is unapproved, or whose definition changed since approval, must be
   approved before it runs — see
   [trust](06-lifecycle-design.md) and
   [requirements](03-requirements.md).
4. Fetch/update shared repository stores in bounded parallelism.
5. Run blocking `pre-create` hooks.
6. Attach worktrees at deterministic paths.
7. Run blocking `post-create` bootstrap per repository.
8. Run workspace-level bootstrap.
9. Create/reuse the terminal runtime workspace.
10. Start optional background services.
11. Mark workspace ready.

Steps 3, 5, 7 and 8 are all failure points, and none of them may be skipped
silently. If approval is declined or a blocking hook fails, the workspace is
**not** marked ready: it stops in `bootstrap-failed` with the failing step
recorded, worktrees left in place for inspection, and background services not
started. That state must be re-runnable, which is what the
[idempotency](06-lifecycle-design.md) requirement buys — retry after fixing the
cause rather than tearing the workspace down and starting again.

A declined approval is a stop, not a downgrade. Continuing without the hook
would produce a workspace that looks ready and is not.

## Example layout

```text
tasks/
└── TASK-123/
    ├── workspace.md
    ├── artifacts/
    ├── .state/
    │   ├── workspace.json
    │   ├── runtime.json
    │   └── health.json
    └── repos/
        ├── backend/
        └── frontend/
```

## Enter

```text
task-open TASK-123
```

The manager:

- resolves workspace path,
- focuses the existing terminal runtime workspace or recreates it,
- allows navigation to workspace root or a repository.

## Status

```text
task-status TASK-123 --json
```

Should expose:

```text
workspace lifecycle state
repositories
branches
dirty state
ahead/behind
bootstrap state
runtime state
background processes
```

Do not infer every fact from terminal text on each invocation if stable metadata can safely record it.

Git-derived facts should still be refreshed from Git when correctness matters.

## Archive

Archive is different from delete.

```text
task-close TASK-123
```

Possible behavior:

- stop disposable processes,
- preserve worktrees and task artifacts or detach them according to policy,
- mark task archived,
- close terminal runtime.

## Reopen

```text
task-reopen TASK-123
```

Reconstruct runtime and rerun only the setup that is no longer valid:

1. Restore worktrees from recorded metadata if archive detached them.
2. Re-check hook approval. Time has passed and the checked-out definition may
   have changed, so approval is re-evaluated exactly as at create.
3. Determine which recorded bootstrap state is still valid, and re-run only the
   invalidated parts as `post-create`.
4. Restart background services and reattach the terminal runtime.
5. Mark ready — or stop in `bootstrap-failed`, on the same terms as create.

Step 3 is the interesting one, and the honest answer is that recorded state is a
claim rather than a fact: archiving does not freeze lockfiles, toolchains, or
caches, and anything the workspace left on disk may have been collected since.
Treating "bootstrap completed" as still true because it was true at archive time
is how reopen produces a workspace that looks ready and is not.

Validity is therefore best derived from observable inputs — lockfile digests,
toolchain versions, the presence of the artifacts a hook was supposed to produce
— rather than from a stored boolean. Where validity cannot be determined, re-run:
the [idempotency](06-lifecycle-design.md) requirement makes an unnecessary re-run
cheap, while a wrongly-skipped one is a broken workspace.

## Purge

Only after safety analysis:

```text
task-purge TASK-123
```

Check:

- dirty worktrees,
- unpushed commits,
- divergent branches,
- unknown upstream state.

Then detach Git worktrees and move disposable filesystem state to trash before eventual GC.
