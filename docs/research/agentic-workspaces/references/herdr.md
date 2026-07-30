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

It should not own the authoritative Git worktree lifecycle when paired with a
workspace manager.

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

## Integrate behind a runtime interface

Should a workspace manager ever grow terminal support, it should arrive behind a
generic runtime interface rather than as a hardcoded integration with one
multiplexer.

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
herdr
directory/no-op
```

Later:

```text
tmux
zellij
```

Expect the real interface to be wider than this sketch. A working integration
generally needs capability negotiation, status reporting, notification and pane
enumeration — not just open/focus/close.

## Initial Herdr integration scope

Keep the first PR small:

- create/reuse terminal workspace for a task path,
- focus it,
- close it when appropriate,
- persist runtime workspace ID,
- optionally run a configured startup command.

Agent layouts and advanced orchestration can come later.
