---
title: "kra"
status: research
scope: references
---

# kra

Repository: `tasuku43/kra` — same author as gion.

Unlike the other pages in this directory, most of what follows has been
**verified against kra's indexed source** rather than recalled from the original
survey. Verified claims are marked; the unverified survey impressions are kept
separately at the end. Still a dated third-party observation — kra may move on.

## Core idea

kra makes the ticket/workspace itself explicit and filesystem-first.

```text
KRA_ROOT/
├── workspaces/<id>/
│   ├── workspace.md          # source of truth for task state
│   ├── .kra.meta.json        # canonical workspace metadata
│   └── repos/<alias>/        # git worktrees
└── archive/<id>/             # closed workspaces
```

Note `archive/` as a sibling of `workspaces/`, and `repos/` as an explicit
subdirectory — gion places repo worktrees directly under the workspace
(`workspaces/<id>/<alias>/`) and has no archive location at all.

Lifecycle states are explicit: `create` → `active` → `close` → `archived` →
`reopen`, with `purge` as terminal from either active or archived.

## Verified: what kra has that gion does not

### Archive and reopen, with worktrees detached

This is the worked answer to gion's hardest open question
([adoption question 4](../gion/adoption-questions.md)). kra does not preserve
worktrees across archive — it **detaches them and keeps the metadata needed to
rebuild**:

`ws close` refreshes `repos_restore` in `.kra.meta.json` from live repository
state, removes each worktree under `repos/<alias>`, deletes the corresponding
local branch from the bare pool, then moves the workspace contents (excluding
`repos/`) to `archive/<id>/`.

`ws reopen` moves the directory back and calls
`recreateWorkspaceWorktreesFromMeta`, which iterates `repos_restore`, ensures
the bare repo exists in the pool, fetches, and recreates each worktree at its
original alias path on the recorded branch. It refuses to reopen if a workspace
branch is already checked out in another worktree.

`repos_restore` records `repo_uid`, `repo_key`, `remote_url`, `alias`, `branch`
and `base_ref` per repository.

**The observation that matters for gion:** those are very nearly the fields
gion's `gion.yaml` already stores per repo entry (`alias`, `repo_key`, `branch`,
`base_ref`). gion's manifest is already a `repos_restore` record — it just has
no state that means "keep this record, drop the worktrees."

### JSON output and a real exit code taxonomy

kra supports `--format json` across core commands, described in its specs as a
deliberate effort to align JSON contracts for automation. The envelope is
`ok` / `action` / `result` / `error`, with a `failures[]` array carrying `code`
and `message` on partial failure. `ws list` also offers `tsv`.

Exit codes are three-valued:

| Code | Meaning |
|---|---|
| 0 | success |
| 2 | usage error — invalid arguments or flag combinations |
| 3 | business or runtime failure |

This is gion's M6 gap, already solved by the same author in a sibling tool.

### A runtime adapter interface

The original survey recommended introducing a generic runtime interface so that
terminal support is not hardcoded to one multiplexer. **kra already has one.**

`cmuxOpenClient` defines the contract — `Capabilities`,
`CreateWorkspaceWithCommand`, `RenameWorkspace`, `SelectWorkspace`, `SetStatus`,
`Notify`, `SendText`, `ListPanes`, `ListWorkspaces`, `Identify` — implemented
concretely by `internal/infra/cmuxctl.Client` and obtained through an
`appcmux.Client` factory, with a `wsOpenClientAdapter` wrapper.

So plugging in another multiplexer is an implementation of an existing
interface, not an architectural change. See [herdr.md](herdr.md), which has been
corrected accordingly.

### Task state inside the workspace

Tasks live in `workspace.md` under a `## Tasks` section as
`### TASK-<nnn> <title>` blocks with a mandatory `status:` line
(`todo`/`doing`/`blocked`/`done`); workspace `task_state` is derived from them.
A machine-centric `workspace.yaml` is supported and takes priority when both
exist.

gion has no equivalent — it manages repository composition, not task content.
Whether it should is a scope question, not a gap.

## Verified: what kra does *not* have

**No lifecycle hooks, and no post-create bootstrap.** Neither `ws create` nor
`ws add-repo` runs user-defined commands. The closest thing is
`ws open --command "<cmd>"`, which sends a startup command to the terminal after
opening an existing workspace — not a creation-time hook, and it runs in the
multiplexer rather than as a tracked lifecycle step.

This matters well beyond kra: the substrate's single mandatory requirement is
unmet by **both** candidate substrates. It is a category-wide gap, not evidence
that gion is behind.

## Verified: where the two converge

kra evaluates repository risk before `close` and before `purge` on an active
workspace, using the same five states gion's `workspacerisk` uses — dirty,
unpushed, diverged, unknown, clean — and prompts for confirmation when any repo
is not clean. `purge` is additionally protected by a "Purge Guard".

Same author, same taxonomy, same safety posture. This is settled design in the
category rather than either tool's invention.

## Unverified: survey impressions

Kept for provenance; not re-checked against source.

gion reads as a declarative reconciler and kra as an operational task workspace
manager. The desired workstation wants both reproducible composition and a
convenient daily task lifecycle. kra's explicit task key is attractive because
task identity survives terminal recreation, agent restarts, branch changes,
archive/reopen, and multi-repository work.
