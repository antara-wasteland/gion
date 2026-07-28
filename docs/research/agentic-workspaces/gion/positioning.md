---
title: "gion's position in the agentic workspace architecture"
status: research
scope: gion
---

# gion's Position

Where gion sits against the [substrate architecture](../substrate/04-architecture-overview.md),
based on the current codebase rather than on aspiration.

## The layer gion occupies

gion is a **task workspace manager**: the middle layer of the substrate diagram.
It owns workspace identity, repository composition, and worktree attachment. It
does not own terminal runtime, and it does not own agent or merge policy.

That boundary is already respected in the code, not merely intended. There is no
model configuration, no API key handling, no commit-message generation, no merge
or PR workflow anywhere in the repository. Everything on the substrate's
[explicitly-out-of-scope list](../substrate/03-requirements.md) is genuinely
absent. This is the single most important fact about gion as a candidate
substrate, and it is worth stating plainly because it is the property that is
hardest to recover once lost.

## The structural match

The substrate argues that the task, not the repository, is the top-level unit.
gion's data model already agrees.

A workspace is a directory containing N repository worktrees
(`docs/spec/core/DIRECTORY_LAYOUT.md`):

```text
GION_ROOT/workspaces/<WORKSPACE_ID>/
  backend/
  frontend/
  .gion/metadata.json
```

A monorepo task has one child. A polyrepo task has several. No special case
exists for either, which is exactly the unified model
[05-monorepo-polyrepo.md](../substrate/05-monorepo-polyrepo.md) asks for.
Presets (`docs/spec/core/PRESETS.md`) give multi-repo composition a name so a
task can be created from a known repository group in one command.

Storage matches too: `<root>/bare/` is a shared bare repository store and every
workspace attaches worktrees from it, so N tasks against one repository cost one
object database rather than N clones.

## The distinguishing choice: declarative reconciliation

gion's defining decision is that workspaces are declared, not imperatively
created. `gion.yaml` is the desired state; `gion plan` diffs it against the
filesystem; `gion apply` reconciles. `gion import` runs the inverse when the
filesystem is the truth.

The substrate notes did not originally ask for this, but it turns out to serve
several substrate requirements at once:

- **Idempotency** — re-running `apply` converges to no changes, which is the
  property [06-lifecycle-design.md](../substrate/06-lifecycle-design.md) wants
  from bootstrap retries, applied one level up.
- **Explicit state** — the manifest *is* the machine-readable declaration of
  what a task should contain, which is much of what the substrate's
  [machine interface](../substrate/03-requirements.md) requirement is reaching
  for.
- **Safety** — `plan` is a mandatory review surface before destruction, rather
  than a confirmation prompt bolted onto a delete command.

The cost is that gion's verbs are reconciliation-shaped (`manifest add`, `plan`,
`apply`) where the substrate's proposed workflow is operation-shaped
(`create`, `open`, `status`, `remove`). That is a surface mismatch over a sound
model, not a model mismatch — see the
[gap analysis](gap-analysis.md) for how much of it is real.

## The functional core

The single most consequential implementation fact about gion is not in this
repository. `go.mod` depends on `github.com/tasuku43/gion-core`, a separate
versioned module with **zero dependencies of its own** — its `go.mod` declares
nothing but `go 1.24.0`.

It holds the pure decision logic:

| Package | Responsibility |
|---|---|
| `planner` | desired-vs-actual inventory diff (`Diff`, `WorkspaceChange`, `RepoChange`) |
| `applyplan` | apply-time semantics: destructive detection, safe rename checks, execution ordering |
| `workspacerisk` | classify repo status into clean/dirty/unpushed/diverged/unknown, aggregate to workspace level |
| `gcplan` | GC fetch planning and merge checks |
| `importplan`, `manifestlsplan` | inventory reconstruction, drift categories |
| `gitparse`, `gitref`, `repostore`, `repospec` | git output parsing, ref and store path normalization |

This repository is the imperative shell: it runs `git`, touches the filesystem,
prompts, and renders. `internal/app/apply/apply.go` consumes
`coreapplyplan.BuildExecution` and executes the result; it does not decide what
the result should be.

Two things follow, and they matter more than any individual feature gap.

**There is already an extension seam.** The architecture is a functional core
with an imperative shell — decisions are pure and testable, effects are isolated.
A new capability does not have to be bolted onto the CLI; it can be expressed as
a decision in the core and an effect in the shell. This is exactly the shape a
hook system wants (see [gap analysis M1](gap-analysis.md#m1--post-create-bootstrap-)).

**The core was extracted for reuse, and is already reused.** `gion-core`'s README
describes it as the *"shared core library for `gion` and `gionx`"* — there is a
second consumer. Upstream has demonstrated, in code, that it will factor generic
primitives out into a dependency-free library rather than keeping them
CLI-specific. For the fork-versus-upstream question this is the strongest signal
available: capabilities implemented generically have a natural home upstream, and
a precedent for getting there.

## Root model

A single global root (`--root` > `GION_ROOT` > `~/gion`) is a default, not a
conceptual limitation. The root can be placed wherever the workstation layout
requires:

```text
workstation/
├── repos/          # durable, inventory-managed repositories
└── tasks/          # gion root: ephemeral task workspaces
```

The distinction that matters is durable repository inventory versus ephemeral
task execution — see
[core insight 6](../substrate/02-core-insights.md). gion is unambiguously the
second, and does not try to be the first.

## What gion would need to add

Summarized here, detailed with evidence in the [gap analysis](gap-analysis.md):

1. **Lifecycle hooks** — the one mandatory substrate requirement with no
   implementation at all.
2. **Machine-readable output** — JSON and meaningful exit codes.
3. **Archive/reopen and trash-before-delete** — currently removal is terminal.
4. **Parallel repository operations during apply** — the polyrepo case is the
   one gion is structurally best at and currently slowest at.

## What gion should not absorb

Terminal runtime ownership, agent assignment, specification state, and merge or
PR policy. If gion ever needs a terminal, it should be through a runtime adapter
interface (see [references/herdr.md](../references/herdr.md)) rather than a
direct integration, so that no single multiplexer becomes a dependency of
reliable worktree creation.
