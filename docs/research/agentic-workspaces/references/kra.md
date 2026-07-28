---
title: "kra"
status: research
scope: references
---

# kra

Repository: `tasuku43/kra`

## Core idea

kra pushes the task-first model further by making the ticket/workspace itself explicit.

Conceptually:

```text
TASK-1234/
├── workspace.md
├── notes/
├── artifacts/
└── repos/
    ├── backend/
    └── frontend/
```

Its lifecycle includes creation/opening, closing/archive, reopening, and eventual purge.

## Where it differs from gion

gion is closer to a declarative reconciler; kra is closer to an operational task
workspace manager. During the survey this read as kra fitting daily operation
better.

That framing should be held loosely now. Much of what made kra look more
operational — archive/reopen, an explicit task lifecycle — is scored honestly
against gion in [gion/gap-analysis.md](../gion/gap-analysis.md), which also finds
that gion's declarative model offers a better home for bootstrap state than
either tool's imperative lifecycle would. The comparison below has not been
re-verified against kra's current source.

The desired workstation needs both ideas:

- reproducible composition,
- convenient daily task lifecycle.

kra's explicit task key is especially attractive because task identity can survive:

- terminal recreation,
- agent restarts,
- branch changes,
- archive/reopen,
- multi-repository work.

## Relationship to cmux

kra has integration with cmux, but terminal integration is not the fundamental workspace model.

That suggests terminal support should be abstracted behind a runtime adapter.

For this workstation, Herdr is the preferred runtime.

## Potential evolution

A strong kra-like substrate could own:

```text
task ID
workspace metadata
repo attachments
archive/reopen
notes/artifacts
runtime mapping
```

while delegating repository bootstrap to lifecycle hooks and terminal/process behavior to Herdr.
