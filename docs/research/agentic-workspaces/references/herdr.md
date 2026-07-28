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

**Corrected.** The original recommendation here was to introduce a generic
runtime interface rather than hardcoding another terminal integration beside
cmux. Checking kra's source shows that interface already exists: `cmuxOpenClient`
(`Capabilities`, `CreateWorkspaceWithCommand`, `RenameWorkspace`,
`SelectWorkspace`, `SetStatus`, `Notify`, `SendText`, `ListPanes`,
`ListWorkspaces`, `Identify`), implemented by `internal/infra/cmuxctl.Client`
and obtained via an `appcmux.Client` factory, with a `wsOpenClientAdapter`
wrapper. See [kra.md](kra.md).

So a Herdr backend is an implementation of an existing seam, not an
architectural change — a much smaller and more upstreamable proposition than the
survey assumed. The open work is whether the interface's shape (which is named
for cmux and carries cmux-flavored methods like `ListPanes`) generalizes cleanly,
or wants a rename and a capability negotiation step first.

The originally sketched shape, for comparison:

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

Note that kra's actual interface is considerably wider than this sketch, which
is the usual outcome: a real integration needs status, notification and pane
enumeration, not just open/focus/close.

## Initial Herdr integration scope

Keep the first PR small:

- create/reuse terminal workspace for a task path,
- focus it,
- close it when appropriate,
- persist runtime workspace ID,
- optionally run a configured startup command.

Agent layouts and advanced orchestration can come later.
