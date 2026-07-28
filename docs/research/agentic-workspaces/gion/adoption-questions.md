---
title: "Open adoption questions for gion"
status: research
scope: gion
---

# Open Adoption Questions

Decisions that shape the [gap analysis](gap-analysis.md) work items. These are
open questions, not proposals — each records what is at stake and which
gion-specific constraint makes it non-obvious.

## 1. Where do hooks live in the declarative model?

Blocks M1, D1, D2 — the adoption blocker.

gion has two candidate homes and they behave differently:

- **In `gion.yaml`** — hooks become desired state, so `plan` can report
  "bootstrap not run" as drift and `apply` can converge it. This is the answer
  that exploits gion's model. It also means hook definitions are rewritten by
  every full-file manifest rewrite (`docs/spec/core/INVENTORY.md`), and it puts
  executable-command strings into the file gion regenerates.
- **In a separate config or repository-owned scripts** — matches
  [core insight 4](../substrate/02-core-insights.md) ("repository-owned bootstrap
  is the stable contract") and keeps the manifest clean, but bootstrap state then
  lives outside the reconciliation loop and `plan` cannot see it.

A hybrid is plausible — the manifest declares *that* a repo bootstraps and the
repository owns *how* (`scripts/worktree-bootstrap`) — but it needs deciding
before either is built.

**Related, unavoidable:** if bootstrap state is drift, `plan` must be able to
determine bootstrap state without running anything, since `plan` is contractually
read-only (`AGENTS.md`). That implies persisted per-repo bootstrap state, most
naturally alongside the existing operation state under `<root>/.gion/state/`.

## 1b. What goes in `gion-core`, and does it land upstream?

gion's logic is split between `github.com/tasuku43/gion-core` (pure decisions,
zero dependencies, shared with `gionx`) and this repository (git, filesystem,
prompts, rendering). Every work item in the gap analysis crosses that line, so
each needs the split decided before implementation:

| Item | Core half | Shell half |
|---|---|---|
| Hooks | is a hook outstanding; ordering vs. worktree attachment | execute, stream output, trust prompt, persist state |
| JSON | already there — the plan types | marshal and emit |
| Exit codes | error taxonomy | map to codes at `main` |
| Parallel apply | already there — bucket ordering in `applyplan` | bounded worker pool |

The follow-on question is release mechanics rather than design: a change that
needs both halves needs a `gion-core` release before this repository can consume
it, so the two-repo round trip should be planned for rather than discovered.

And the strategic one: since these are generic capabilities in a library
explicitly built for reuse, the default should be upstreaming rather than
carrying them on a fork. Worth deciding deliberately — including how much
prototyping happens here first — rather than by accident.

## 2. Hook semantics

Still unresolved from [06-lifecycle-design.md](../substrate/06-lifecycle-design.md),
independent of where hooks live:

- exact event set (`pre-create`, `post-create`, `post-create-bg`, `pre-remove`,
  `post-remove`, and whether `enter` events exist at all given `giongo`)
- blocking versus background semantics and how background processes are tracked
  and cancelled
- retry semantics on failure
- the environment contract — the substrate proposes `TASK_ID`, `WORKSPACE_PATH`,
  `REPO_NAME`, `WORKTREE_PATH`, `BRANCH`, `INDEX`; gion's names would be
  `WORKSPACE_ID` and `ALIAS`, and the mapping should be settled once
- workspace-level versus repo-level hooks (gion has both scopes natively; most
  repository-first references have only one)
- trust model and where approvals persist
- how a failed blocking hook is represented in `plan` output

## 3. What does the `Apply` section print when hooks run?

A gion-specific constraint the substrate never had to consider. `AGENTS.md` and
`docs/spec/ui/UI.md` require the `Apply` section to summarize at workspace/repo
level and explicitly *not* dump raw commands. A `pnpm install` hook produces a
lot of output that some users will want and the UI contract forbids by default.

Options: a summarized step line with output routed to the existing `--debug` log
(`<root>/logs/debug-YYYYMMDD.log`), or an opt-in verbose flag. This should be
settled as part of the hook spec rather than discovered during implementation.

## 4. How is an archived workspace represented in `gion.yaml`?

Blocks D3. **Narrowed** — kra's implementation answers the semantic half (see
[references/kra.md](../references/kra.md)): archived worktrees are detached and
metadata is the restore authority, not preserved in place. And gion's manifest
repo entries already carry the fields kra's `repos_restore` uses.

So the remaining question is narrower than it was: should archived be a third
state in `planner`'s diff — materialized, archived, absent — or something
outside the manifest entirely? A third state keeps one source of truth and
avoids kra's directory move (gion already separates desired state from layout),
but it touches the core diff model that everything else relies on.

gion's diff is two-state: in the manifest and on disk (converged), in one but not
the other (add/remove). An archived workspace is neither, and if it is simply
left in the manifest with its directory gone, `apply` will recreate it — while
removing it from the manifest makes `apply` delete the worktrees archiving was
supposed to preserve.

So archive needs a manifest-level state before it needs a command. The
attached-versus-detached sub-question is effectively settled by kra's precedent
(detach, restore from metadata); what remains open is whether the choice should
ever depend on policy or workspace age, or whether one behavior is enough.

Note that `resume` already exists as a `mode` value in
`docs/spec/core/METADATA.md` without a lifecycle behind it — worth deciding
whether archive/reopen adopts it or supersedes it.

## 5. Should `status` be a command or a flag on `plan`?

Blocks part of M2 and interacts with M6.

`gion plan` already computes per-repo risk (dirty, unpushed, diverged, sync
state, changed files) for removal previews. A `status` command would compute
nearly the same data for a different question: not "does the filesystem match the
manifest" but "what is the state of this workspace's work."

Sharing the computation is clearly right. Whether that surfaces as
`gion status [--json]` or as an extension of `plan` is open, and the answer
should probably be driven by what an agent wants to call, since agents are the
primary consumer.

## 6. Scope of the first JSON output

Blocks M6. **Mostly answered** — the schema question should not be reopened.
kra already ships `--format json` with an `ok`/`action`/`result`/`error`
envelope, a `failures[]` array of `code`/`message`, and exit codes `0`/`2`/`3`
(success / usage / runtime). Same author, same workstation. gion should mirror
that rather than invent a second convention for an agent to learn.

What is still open is scope and spelling:

- Which commands first? `plan` and a future `status` are what an agent most
  needs to read; committing every command at once is a large stable surface.
- `--format json` (kra's spelling) or `--json`? Matching kra is the obvious
  default.
- Does `giongo` participate, or is `--print` sufficient for the picker?

Exit codes are independent and cheaper, and could land first — both
`cmd/gion/main.go` and `cmd/giongo/main.go` currently `os.Exit(1)` on every
error path, so adopting `0`/`2`/`3` is purely additive.

## 7. Should the shared bare store be gion's own?

The substrate warns against duplicate Git object stores
([07-storage-performance.md](../substrate/07-storage-performance.md)). gion owns
`<root>/bare/`. A workstation that also keeps durable repository inventory
elsewhere would hold two object stores for the same repositories.

Since `--root` is freely placeable, this may be a layout convention rather than a
code change — but it should be written down as guidance either way.

## 8. `focus` for monorepo tasks

Small, additive, and unresolved: should a workspace repo entry carry a `focus`
path so that a monorepo task can record which package it touches, for navigation
(`giongo` landing directory), scoped bootstrap, agent working directory, and
status presentation?

It must not create extra Git worktrees — Git's unit is the repository.

## 9. Terminal runtime: adapter, or out of scope entirely?

[references/herdr.md](../references/herdr.md) argued for a generic
`WorkspaceRuntime` interface rather than a hardcoded multiplexer integration.
That argument has been overtaken: kra already has such an interface
(`cmuxOpenClient`, with a concrete `cmuxctl.Client` behind a factory), so a
Herdr backend there is an implementation against an existing seam rather than an
architectural change.

The prior question for gion is unchanged and more pointed for it: does terminal
runtime belong in gion *at all*, given that `gion shell init` already solves
navigation without owning any session state — and given that the sibling tool
which does own a runtime already has the adapter?

Leaving it out keeps gion smaller and keeps the substrate's ownership boundary
clean. Taking it on means persisting a runtime workspace ID in metadata and
answering the recovery cases: runtime restarts, metadata survives but the
terminal workspace is gone, or the terminal workspace exists but the workspace
was removed. The task workspace should remain authoritative in every case.
