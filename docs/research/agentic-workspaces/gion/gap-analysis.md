---
title: "Gap analysis: substrate requirements vs. gion"
status: research
scope: gion
---

# Gap Analysis

Every requirement from
[substrate/03-requirements.md](../substrate/03-requirements.md) scored against
the gion codebase, with the evidence that produced each score.

Assessed against `18e1fd2` (2026-07), which is also `tasuku43/gion` `main` at the
time of writing — so these findings describe upstream, not fork-local drift, and
the work items below are upstreamable rather than fork-only.

Scores are claims about code that exists today, not about intent. Where a
requirement is met by a different mechanism than the substrate notes imagined,
that is recorded as met, with the mechanism named.

**Legend** — ✅ met · 🟡 partial · ❌ missing

## Architectural context

gion is a **functional core with an imperative shell**, and the core lives in a
separate zero-dependency module, `github.com/tasuku43/gion-core`: `planner`
(inventory diff), `applyplan` (destructive detection, execution ordering),
`workspacerisk` (repo state classification and aggregation), `gcplan`,
`importplan`, `manifestlsplan`, plus git parsing and path normalization
helpers. This repository runs git, touches the filesystem, prompts, and renders.

This shapes almost every finding below, so it is stated once here:

- **The plan is already a structured value in a reusable library.** `--json`
  (M6) is a marshaling layer over `planner.WorkspaceChange` and friends, not new
  data plumbing.
- **The risk taxonomy is a pure state machine, not CLI code.** M3's states come
  from `workspacerisk.ClassifyRepoStatus` and `AggregateForState`.
- **There is a natural seam for new capabilities** — decide in the core, execute
  in the shell — which is the shape hooks want (M1).
- **Generic primitives have a route upstream.** `gion-core` is described as the
  shared library for `gion` *and* `gionx`; upstream already factors reusable
  logic out of the CLI. See [positioning](positioning.md#the-functional-core).

## Upstream's own known gaps

`docs/guides/USECASES.md` rates gion's UX per use case and ends with a
"Known Gaps / Improvement Ideas" list. It is worth reading before proposing
anything, because upstream has already named several of these — and it includes
an explicit **"Human + Agent Co-use"** section, so the agent use case is not a
foreign concern being introduced from outside.

| Upstream's stated gap | Relation to this analysis |
|---|---|
| No JSON/machine-readable output; agents must parse human text | **M6**, independently identified |
| `gion doctor` checks a narrow set; misses orphan worktrees, branch conflicts, stale artifacts | **D5**, and broader than what this analysis first scored |
| Bare store freshness depends on manual `git fetch`; no auto/update flow | **Missed here** — see M5 below |
| `--review` workspaces need manual fetch to pull newer PR updates; no refresh command | **Missed here** — same root cause |
| `manifest ls` needs a drift model that stays stable and understandable | Not a substrate requirement; upstream's own concern |

Two consequences.

First, the two headline gaps in this document are not outside criticism — the
maintainer wrote them down first. That is the strongest available evidence for
the [fork-or-upstream](#fork-or-upstream) conclusion, and stronger than any
inference from the architecture.

Second, this analysis missed two real gaps by scoring the substrate's
requirement list without reading upstream's own. Both are recorded below.

## Summary

| # | Requirement | Score | One-line assessment |
|---|---|---|---|
| **Mandatory** ||||
| M1 | Post-create bootstrap | ❌ | No hook system of any kind exists |
| M2 | Ergonomic CLI | 🟡 | Reconciliation verbs cover creation and removal; no `status`, no `exec` |
| M3 | Safe cleanup | ✅ | Risk taxonomy, plan-before-destroy, resumable removal state |
| M4 | Monorepo and polyrepo | ✅ | Workspace-of-N-worktrees is the native model; no `focus` |
| M5 | Efficient storage | ✅ | Shared bare store at `<root>/bare/` — but nothing refreshes it |
| M6 | Machine interface | ❌ | No JSON, no exit code taxonomy |
| **Strongly desired** ||||
| D1 | Hook trust/approval | ❌ | Blocked on M1 |
| D2 | Blocking/background hooks | ❌ | Blocked on M1 |
| D3 | Archive/reopen | ❌ | Removal is terminal |
| D4 | Trash-before-delete | ❌ | Deletion is immediate |
| D5 | Doctor/repair | 🟡 | Checks are narrow *and* `--fix` is a declared no-op |
| D6 | Path templates | ❌ | Layout hardcoded |
| D7 | Custom subcommands | ❌ | Closed dispatch switch |
| D8 | Shell navigation protocol | ✅ | Action-file protocol, already integrated with apply |
| D9 | Parallel operations | 🟡 | Bounded pool in `gc` fetch only; `apply` is serial |
| **Out of scope** ||||
| X | LLM/merge/PR/model policy | ✅ | Genuinely absent |

Seven of nine substrate gaps trace back to two absent subsystems: **hooks**
(M1, D1, D2) and **machine output** (M6). The rest of the architecture is
already in place.

Two further gaps come from upstream's own list rather than the substrate's, and
are recorded under M5 and D5: **nothing refreshes the bare store**, and
**`doctor`'s checks are narrow as well as unrepaired**. See
[upstream's own known gaps](#upstreams-own-known-gaps).

---

## Mandatory requirements

### M1 · Post-create bootstrap ❌

**Required:** a newly created worktree runs deterministic setup (`uv sync`,
`pnpm install --frozen-lockfile`, submodule init, `direnv allow`,
`./scripts/worktree-bootstrap`), receiving `TASK_ID`, `WORKSPACE_PATH`,
`REPO_NAME`, `WORKTREE_PATH`, `BRANCH`, `INDEX`.

**Found:** nothing. There is no hook subsystem, no hook configuration in
`gion.yaml` (`docs/spec/core/INVENTORY.md` defines only `version`, `workspaces`,
`presets`), and no execution point after worktree attachment.
`internal/app/apply/apply.go:231` (`applyRepoAdds`) attaches the worktree and
returns; the next thing that happens is output rendering.

Grepping for `hook` in `internal/` returns only `applyManifestMutation`'s
render callbacks (`ShowPrelude`, `RenderInfoBeforeApply` in
`internal/cli/manifest_mutation.go`) — an unrelated internal presentation
concept that happens to share the word.

**Assessment.** This is the one requirement the substrate marks as
non-negotiable, and it is the one with zero implementation. Everything else in
this document is refinement; this is the adoption blocker. A workspace gion
creates is a checked-out worktree and nothing more — the user still has to enter
it and run project setup by hand, which is precisely the operational gap that
motivated the whole survey.

Worth keeping in proportion: **kra does not have hooks either**. Neither
`ws create` nor `ws add-repo` runs user-defined commands, and its
`ws open --command` sends a startup command to the terminal rather than running
a tracked lifecycle step (see [references/kra.md](../references/kra.md)). This
is a category-wide gap, not gion falling behind a sibling — which makes it a
capability worth building well and upstreaming rather than a deficit to catch up
on.

**Shape of the fix.** The declarative model gives this a natural home that
imperative tools do not have: bootstrap state belongs in the reconciliation
loop, not beside it. A workspace whose blocking hooks have not succeeded is
*drift*, which means `plan` can report it and `apply` can converge it, and a
failed bootstrap becomes a re-runnable state rather than a wedged directory.
That is a genuinely better answer than the one the substrate notes sketched, and
it is available only because gion is declarative.

The core/shell split says where the pieces go, and the split is clean:

- **`gion-core`** decides *whether* a hook needs to run — a bootstrap-state
  input to `planner.Diff`, so "bootstrap not run" becomes a `RepoChange` kind
  like any other, and a pure `applyplan` rule for ordering hooks against
  worktree attachment. All of this is testable without executing anything.
- **`gion`** executes the hook, streams its output, enforces the trust
  prompt, and persists the resulting state.

Notably this keeps `plan` read-only for free: deciding that bootstrap is
outstanding is a pure function of recorded state, so `plan` never has to run a
hook to report that one is pending. It also puts the reusable half in the
library that already serves two binaries, which is the upstreamable shape.

Design questions that need resolving first are in
[adoption-questions.md](adoption-questions.md).

### M2 · Ergonomic CLI 🟡

**Required:** `create`, `open`/`enter`, `list`, `status`, `exec`, `remove`,
`cleanup` — cheap enough for muscle memory, and all scriptable.

**Found:** the dispatch table in `internal/cli/app.go:85-101` is `init`,
`doctor`, `repo`, `manifest`, `plan`, `import`, `apply`, `completion`, `shell`.
Mapping to the requirement:

| Substrate verb | gion | Notes |
|---|---|---|
| create | `gion manifest add` | Runs `apply` by default, so it is one command |
| open / enter | `giongo` | Interactive picker; `--print` for scripting |
| list | `gion manifest ls`, `gion repo ls` | |
| status | — | **Missing.** `gion plan` shows drift, not workspace health |
| exec | — | **Missing.** No way to run a command across a workspace's repos |
| remove | `gion manifest rm` | |
| cleanup | `gion manifest gc` | Conservative bulk removal |

**Assessment.** The naming is reconciliation-shaped rather than
operation-shaped, and `gion manifest add --repo X TASK-1` is more typing than
`create TASK-1 X`. That is a real ergonomic cost but a shallow one — aliases or
top-level shorthands could close it without touching the model.

The two genuine holes are `status` and `exec`. `status` matters most for agents:
[08-task-workspace-workflow.md](../substrate/08-task-workspace-workflow.md)
wants one call returning lifecycle state, branches, dirty state, ahead/behind and
bootstrap state, and `gion plan` answers a different question (does the
filesystem match the manifest). The risk-summary machinery `plan` already builds
for removals is most of the data a `status` command would need, so this is
closer than the ❌ items.

### M3 · Safe cleanup ✅

**Required:** distinguish clean/integrated, dirty, unpushed, diverged, unknown,
stale, archived; destructive operations fail safely by default.

**Found:** the taxonomy is a pure state machine in the core, not ad-hoc CLI
formatting. `workspacerisk.ClassifyRepoStatus` maps a repo's status to
`Clean`/`Dirty`/`Unpushed`/`Diverged`/`Unknown` with explicit precedence —
error, dirty, and detached/missing-HEAD/no-upstream all resolve before
ahead/behind is consulted, and ahead+behind is `Diverged` rather than
`Unpushed`. `AggregateForState` then rolls repo states up to a workspace risk
with the conservative ordering dirty > unknown > diverged > unpushed > clean, so
one bad repo governs the whole workspace.

On top of that, `docs/spec/commands/plan.md` — removal plans print a per-repo
risk summary plus ahead/behind sync state, changed-file counts and the
modified/untracked/conflicted file list. `docs/spec/commands/manifest/gc.md`
excludes any workspace containing a repo in dirty, unpushed, diverged or unknown
state.

Safety defaults are strong and specified, not incidental:

- destructive actions always prompt; `--no-prompt` **errors** rather than
  proceeding (`docs/spec/commands/apply.md`)
- `plan` is read-only with no side effects (`AGENTS.md`)
- removal persists resumable state at
  `<root>/.gion/state/operations/workspace-remove/<id>.json`, and a subsequent
  `apply` fails fast if an unfinished removal exists
- if the process cwd is inside a workspace being removed, gion shifts cwd to
  `<root>` first and emits a shell action to move the parent shell

**Assessment.** This exceeds what the substrate asked for, and it is the
requirement gion implements most rigorously. The classification being pure and
centrally defined means every consumer — `plan`, `gc`, `manifest ls` — agrees on
what "unpushed" means, which is exactly the property a safety taxonomy needs and
the one most easily lost to per-command reimplementation.

The interrupted-removal state file answers a failure mode the requirements list
did not anticipate: a removal killed halfway leaves a recorded, inspectable state
instead of a half-deleted workspace. Only `stale` and `archived` are
unrepresented, and both are consequences of D3 rather than cleanup defects.

### M4 · Monorepo and polyrepo ✅

**Required:** one task representing either a single monorepo worktree or several
polyrepo worktrees, without changing the workflow model.

**Found:** exactly the required layout
(`docs/spec/core/DIRECTORY_LAYOUT.md`). A workspace is a directory of aliased
repo worktrees; `repos` is a list in `gion.yaml` with no cardinality special
case. Presets name reusable repository groups (`docs/spec/core/PRESETS.md`), and
`gion manifest add --preset` creates a multi-repo workspace in one command with
per-repo branch and base prompts.

**Assessment.** Met, and it is gion's strongest structural advantage over
repository-first managers. The only shortfall is the `focus` idea from
[05-monorepo-polyrepo.md](../substrate/05-monorepo-polyrepo.md) — a monorepo
task that touches one package has no way to record which package, for navigation
or scoped bootstrap. That is a small additive field, not a model problem, and it
correctly does not imply extra worktrees.

### M5 · Efficient storage ✅

**Required:** avoid full clones per task; share Git object stores.

**Found:** `<root>/bare/` is a shared bare repo store
(`internal/infra/paths/roots.go`), keyed as `<host>/<owner>/<repo>.git`. Every
workspace repo entry references a `repo_key` into that store and is materialized
as a worktree. Ten tasks against one repository share one object database.

**Assessment.** Met, and it is architecturally explicit rather than emergent —
the store is a named directory with its own commands (`gion repo get/ls/rm`).
Shared *package* caches (pnpm, uv, Cargo) are out of gion's hands and belong to
whatever bootstrap runs under M1.

**Caveat: the store goes stale, and nothing refreshes it.** Storage efficiency
is met; storage *freshness* is not. `gion repo get` does not fetch when the bare
store already exists (`docs/guides/USECASES.md` rates this "Good … not ideal for
updating"), and upstream lists the absence of an auto/update flow as a known gap.
`manifest gc` is the only command that fetches, and only for its own merge
checks.

This matters more for agents than for humans, and it is a substrate requirement
rather than a nicety: step 3 of
[the task workspace workflow](../substrate/08-task-workspace-workflow.md) is
"fetch/update shared repository stores in bounded parallelism". gion does the
attach but not the update. The consequence is silent: a workspace created
against a stale store cuts its branch from a stale `origin/main` and nothing
reports that it happened. A human notices when their branch is oddly behind; an
agent creating many workspaces unattended does not.

Upstream names the same root cause a second time for `--review` workspaces,
where pulling newer PR commits also requires a manual fetch. A single
`--fetch`/refresh capability would address both, and it pairs naturally with
the parallel-fetch work in D9 — bounded concurrent fetching is exactly what
`manifest gc` already implements.

### M6 · Machine interface ❌

**Required:** stable exit codes, JSON output, non-interactive flags,
deterministic paths, explicit state — "agents should not need to scrape pretty
terminal output."

**Found:** mixed, but failing on the two that matter.

| Sub-requirement | State |
|---|---|
| Deterministic paths | ✅ `<root>/workspaces/<id>/<alias>`, fixed |
| Non-interactive flags | ✅ `--no-prompt` throughout, `--root` |
| Explicit state | 🟡 `.gion/metadata.json` + operation state files |
| JSON output | ❌ |
| Stable exit codes | ❌ |

`docs/spec/README.md` states it outright: *"human-readable text only in the
current MVP; JSON output is future work."* The only `encoding/json` use in the
CLI is `internal/cli/issue_review_helpers.go`, parsing `gh` output — gion
consumes JSON but does not emit it. Exit codes are binary: both `cmd/gion/main.go:22`
and `cmd/giongo/main.go:22` do `os.Exit(1)` on any error, so a caller cannot
distinguish "manifest invalid" from "git failed" from "user declined" without
parsing English error text.

**Assessment.** This is the second blocker, and it bites hardest exactly where
gion is otherwise strongest. gion is heavily invested in a rich sectioned TUI
(`docs/spec/ui/UI.md`, `internal/ui/`) — good for humans, useless for the agents
the substrate is being built to serve. An agent that wants to know what `apply`
would do has to scrape `Plan` section text.

It is also the cheapest gap to close, which makes the current state more
frustrating rather than less. The data an agent wants already exists as plain
structured values in a zero-dependency library: `planner.WorkspaceChange` and
`RepoChange` with typed `Kind` enums, `applyplan.Execution` with the five
ordered execution buckets, `workspacerisk.RepoState` and `WorkspaceRisk`.
`internal/app/manifestplan/types.go` is a thin local alias over
`coreplanner`. Text rendering already happens at the edge, in `internal/cli`.

So `--json` is a second renderer over types that exist, are pure, are already
versioned for reuse across two binaries, and carry no formatting concerns —
not new data plumbing. Exit codes are independent of it and cheaper still.

**And the design work is already done, by the same author.** kra supports
`--format json` across its core commands with an `ok` / `action` / `result` /
`error` envelope and a `failures[]` array carrying `code` and `message`, plus a
three-valued exit taxonomy: `0` success, `2` usage error, `3` business or
runtime failure. Its specs describe this as a deliberate effort to align JSON
contracts for automation.

That reframes M6 from a design problem to an adoption one. gion should not
invent a schema; it should mirror kra's, so that an agent driving both tools
sees one convention. It also removes the main argument for deferring: the
envelope, the error-code discipline and the exit codes have already been
specified and shipped somewhere the maintainer controls.

---

## Strongly desired

### D1 · Hook trust/approval ❌ · D2 · Blocking/background hooks ❌

Both blocked on M1. Worth designing together with it rather than after: trust
model and blocking/background semantics change the hook configuration schema, and
retrofitting them costs a breaking change to `gion.yaml`. See
[06-lifecycle-design.md](../substrate/06-lifecycle-design.md) for the semantics
and [adoption-questions.md](adoption-questions.md) for what is unresolved.

### D3 · Archive/reopen ❌

No archive concept exists — `archive` appears nowhere in `internal/` or
`docs/spec/`. Removal is the only exit from a workspace, and it is terminal.

One adjacent hook exists: `resume` is a valid `mode` value in
`docs/spec/core/METADATA.md`, which suggests re-entering existing work was
anticipated in the metadata model even though no lifecycle implements it.

Note the interaction with the declarative model: an archived workspace is
neither present-and-desired nor absent-and-undesired, so it does not fit the
current two-state diff. Archive needs a manifest representation before it needs
a command, or `apply` will simply reconcile archived workspaces out of existence.
This remains the requirement whose design is most entangled with gion's core
model.

**kra has a worked answer, and gion is closer to it than expected.** kra's
`ws close` refreshes a `repos_restore` record from live state, removes the
worktrees, deletes the local branches from the bare pool, and moves the
workspace to a sibling `archive/` directory; `ws reopen` rebuilds the worktrees
from that record. So archived worktrees are **detached, with metadata as the
restore authority** — the "reduced to lighter metadata" option, not
"remain attached."

`repos_restore` stores `repo_uid`, `repo_key`, `remote_url`, `alias`, `branch`
and `base_ref`. gion's `gion.yaml` repo entries already store `alias`,
`repo_key`, `branch` and `base_ref` — **gion's manifest is already a
`repos_restore` record.** It has everything needed to reconstruct a workspace;
what it lacks is a state meaning "keep this record, do not materialize
worktrees."

That makes the likely shape a workspace-level status in `gion.yaml` that
`planner` treats as a third desired state — materialized, archived, absent —
rather than a new storage location or a parallel metadata file. gion would not
need kra's directory move, because gion already separates desired state from
filesystem layout. Still a real change to the diff model, and still the item to
design most carefully, but no longer an open-ended question.

### D4 · Trash-before-delete ❌

No trash mechanism. `manifest rm` and `apply` remove worktrees and directories
directly.

Partial mitigation already exists: the risk analysis in M3 makes accidental
destructive removal much less likely in the first place, which is the higher-value
half of the protection. Trash adds a recovery window after a *correct-looking*
decision turns out to be wrong, plus the perceived-performance benefit of
deferring large deletes to GC.

### D5 · Doctor/repair 🟡

`gion doctor` exists and diagnoses well (`docs/spec/commands/doctor.md`): root
layout validation, workspace scanning with per-repo warnings, repo stores with
missing or URL-less `origin` remotes, and `--self` for environment diagnostics
(git present, version ≥ 2.20.0, OS caveats).

The gap is repair. The spec is explicit that `--fix` *"currently performs the
same checks and returns the list of issues; no automatic fixes are applied yet
(the `fixed` list remains empty)"*. The flag exists and does nothing, which is
worse than absent for an agent that trusts it.

Upstream scores this the same way and goes further: `docs/guides/USECASES.md`
rates the repo store health check "Fair (narrow check set)" and lists the missed
categories — **orphan worktrees, branch conflicts, and stale workspace
artifacts**. So the gap is both halves: the checks are narrow *and* the repair
is absent. Those three categories are also precisely the failure modes an
unattended agent produces most, which raises this above the convenience tier it
was first placed in.

### D6 · Path templates ❌

`internal/infra/paths/roots.go` hardcodes `<root>/bare` and `<root>/workspaces`;
worktree paths are `<root>/workspaces/<id>/<alias>` throughout. Only the root is
configurable (`--root` > `GION_ROOT` > `~/gion`).

Note the tension with M6's deterministic-paths requirement: gion's rigidity is
partly *why* its paths are predictable to agents. Templates should be treated as
a low-priority convenience, and any design should keep the default layout
mechanically derivable.

### D7 · Custom subcommands ❌

`internal/cli/app.go:85-101` is a closed `switch`. There is no `gion-<name>`
PATH lookup or alias mechanism, so unrecognized commands cannot be delegated.

Low priority: a shell wrapper covers most of what this buys, and `exec` (M2)
would cover more of the remainder.

### D8 · Shell navigation protocol ✅

Fully present and better integrated than the requirement asked for.
`gion shell init [bash|zsh]` (`docs/spec/commands/shell/init.md`) prints a
wrapper that creates a temp action file, exports `GION_SHELL_ACTION_FILE`, runs
`command gion`, and applies the action **only on success**. `giongo --print`
gives the scriptable path-only path.

The integration detail worth noting: `apply` uses this protocol for correctness,
not just convenience — removing the workspace you are standing in emits a cwd
change to `<root>`, and explicitly does not emit an action on failure.

### D9 · Parallel operations 🟡

The substrate is specific here: bounded concurrency, cancellation, clear errors,
idempotent retry — and it matters most for polyrepo task creation.

**Found:** two pieces of concurrency, neither on the critical path.

- `internal/cli/manifest_gc.go:107-135` — a proper bounded worker pool
  (4 workers, job channel, buffered results, per-repo error aggregation) for
  fetching repos during GC. This is the pattern the substrate describes.
- `internal/infra/prefetcher/prefetcher.go` — speculative background fetch
  started during interactive prompting, so the repo is warm by the time the user
  finishes answering.

**But `apply` is entirely serial.** `internal/app/apply/apply.go` is plain `for`
loops over removals, updates, renames and adds; `applyWorkspaceAdd:107` iterates
repo entries one at a time, and `applyRepoAdds:231` does the same. Creating a
three-repo workspace fetches and attaches three repos sequentially.

**Assessment.** The right pattern is already in the codebase — it just is not
applied where the substrate says it matters most. This is the gap with the best
effort-to-value ratio: the polyrepo case is the one gion is uniquely good at
modelling and currently slowest at executing, and `gc` provides a working
template to copy.

The core/shell split helps here too. `applyplan.BuildExecution` already
partitions changes into five ordered buckets (removals, update-removals,
renames, adds, update-adds); ordering *between* buckets is a correctness
constraint, but repo operations *within* a bucket are independent. Parallelism
is therefore a shell-side change with the safety argument already encoded in the
core — no reasoning about ordering has to be redone to make it safe.

---

## Out of scope — correctly absent ✅

The substrate explicitly excludes LLM-generated commit messages, model API keys,
task decomposition, autonomous merging, PR policy, CI decision policy, and
specification ownership.

None of these appear anywhere in gion. There is no model configuration, no API
key handling, no merge command, no PR workflow. `--review` and `--issue` modes
read GitHub metadata via `gh` to *derive workspace identity* — a workspace ID and
a branch name — and stop there; they do not manage the PR.

This is worth stating as a finding rather than an absence. The substrate's
central warning is that infrastructure gets coupled to policy and becomes
unreplaceable, and it is much easier to add hooks to a tool that has stayed out
of policy than to extract policy from a tool that has not.

---

## Conclusion

gion's **architecture** matches the substrate closely: task-first composition,
shared object storage, safe cleanup, deterministic paths, and a disciplined
refusal to absorb agent or merge policy. Where it diverges — declarative
reconciliation instead of imperative operations — the divergence generally
serves the substrate's goals rather than fighting them, and in the M1 case
offers a better answer than the substrate proposed.

gion's **interfaces** are where it falls short: no way to extend the lifecycle
(M1/D1/D2) and no way for a machine to read the output (M6).

That is a favorable shape for adoption. Both gaps are additive — neither
requires reopening the core model, and both are the kind of generically useful
primitive worth building properly rather than working around. Archive (D3) is
the exception and should be designed against the manifest model deliberately.

## Fork or upstream?

The substrate's [fork strategy](../substrate/03-requirements.md) says to fork
only when the architectural *unit* differs, and to prefer upstreamable
primitives otherwise. On this evidence, nothing here justifies a fork.

gion already is task-first, already composes polyrepo workspaces, already stays
out of policy, and — via `gion-core` — already factors reusable logic into a
dependency-free library serving more than one binary. The gaps are missing
capabilities within an architecture that agrees with the substrate, not
disagreements about what a workspace is.

Every top work item also happens to be generically useful rather than
workstation-specific: lifecycle hooks, JSON output, exit codes and parallel
apply are things any gion user wants, and each splits cleanly into a pure
`gion-core` half and a `gion` execution half. That is the profile the substrate
describes as upstreamable, and the fork's `main` is currently identical to
upstream's, so there is no divergence to reconcile first.

The decisive evidence is that **upstream already wants most of this**.
`docs/guides/USECASES.md` independently lists the JSON gap, the narrowness of
`doctor`, and the missing store-refresh flow as known gaps, and dedicates a
section to "Human + Agent Co-use". These are not outside requirements being
pushed onto a project that does not share them; they are the maintainer's own
backlog, arrived at separately. Contributing against a list someone has already
written is a materially different proposition from arguing for a new direction.

The honest counter-argument is velocity: upstreaming is slower than shipping
locally, and the hook design (question 1 in
[adoption-questions.md](adoption-questions.md)) touches the manifest schema,
which is the kind of decision an upstream maintainer will rightly want to own.
Prototyping on the fork to answer the open design questions is reasonable;
keeping the result there permanently is not, on this evidence.

Suggested order, by blocker status and effort-to-value. Items marked ★ are on
upstream's own known-gaps list, so they need no case made for them:

1. **M1 + D1 + D2** — lifecycle hooks with trust and blocking/background
   semantics, designed as one unit. The adoption blocker.
2. ★ **M6** — mirror kra's `--format json` envelope and `0`/`2`/`3` exit codes
   over the existing plan types. The agent-facing blocker, and the design is
   already settled in a sibling tool.
3. ★ **M5-freshness + D9** — a fetch/refresh flow for the bare store and for
   `--review` workspaces, built on bounded parallel fetching, then parallel
   `apply` using the same pool. These share machinery and `manifest gc` already
   has the pattern; best combined effort-to-value.
4. **M2** — `status` and `exec`.
5. ★ **D5** — widen `doctor` to orphan worktrees, branch conflicts and stale
   artifacts, and either make `--fix` real or remove it until it is. Higher
   priority than first scored, because those are the failure modes unattended
   agents actually produce.
6. **D3** — archive/reopen. Semantics are settled by kra's precedent; what
   remains is whether archived becomes a third `planner` state.
7. **D4, D6, D7, M4-focus** — convenience; no rush.
