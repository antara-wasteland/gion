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
3. Fetch/update shared repository stores in bounded parallelism.
4. Attach worktrees at deterministic paths.
5. Run blocking post-create bootstrap per repository.
6. Run workspace-level bootstrap.
7. Create/reuse the terminal runtime workspace.
8. Start optional background services.
9. Mark workspace ready.

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

Reconstruct runtime and rerun only required setup.

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
