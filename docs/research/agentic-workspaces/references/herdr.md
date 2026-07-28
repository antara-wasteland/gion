---
title: "Herdr integration"
status: research
scope: references
---

# Herdr Integration

Herdr is the preferred terminal multiplexer/runtime layer.

## Ownership

Herdr should own:

- terminal workspaces,
- tabs,
- panes,
- running commands,
- coding-agent processes,
- focus/navigation,
- process/output status.

It should not own the authoritative Git worktree lifecycle when paired with kra/Gion.

Avoid two managers both believing they created and own the same worktree.

## Desired mapping

```text
TASK-123
├── task workspace
│   ├── backend/
│   └── frontend/
│
└── Herdr workspace
    ├── agent tab
    ├── services tab
    └── tests tab
```

Persist the Herdr workspace ID in task metadata.

Do not rely only on display labels.

## kra PR direction

Rather than hardcoding another terminal integration beside cmux, introduce a generic runtime interface.

Conceptually:

```go
type WorkspaceRuntime interface {
    Available(ctx context.Context) bool
    Open(ctx context.Context, workspace Workspace) error
    Focus(ctx context.Context, id string) error
    Close(ctx context.Context, id string) error
}
```

Implementations could include:

```text
cmux
herdr
directory/no-op
```

Later:

```text
tmux
zellij
```

## Initial Herdr integration scope

Keep the first PR small:

- create/reuse terminal workspace for a task path,
- focus it,
- close it when appropriate,
- persist runtime workspace ID,
- optionally run a configured startup command.

Agent layouts and advanced orchestration can come later.
