# Snapshot completeness / publish guard for subsystem-edges

Status: open (deferred)
Type: task

## Problem

`xo-cmake/etc/xo/subsystem-edges` is a checked-in, installed snapshot of the
inter-subsystem dependency graph. It is the ONLY source of the graph for a
downstream consumer using `xo-build --clone --with-deps <lib>` to selectively
adopt a subset of the (dozens of) XO repos: that consumer has no repos cloned
yet, so it cannot regenerate edges by configuring — it must trust the published
snapshot. In-tree dev builds never read it (they use the fresh per-configure
`.build/xo-subsystem-edges.txt`), so this risk is downstream-only.

The snapshot is written by `xo_emit_dependency_edges()` from whatever edges the
current configure saw. It can be published incomplete/stale, silently:

- **Incomplete configure:** the generated graph is only complete if ALL optional
  flags were set at configure time. Re-snapshot from a partial configure →
  smaller graph, written without warning.
- **Lag:** not re-snapshotting after a dependency change → snapshot trails the code.

Downstream symptom of a missing edge: `--clone` under-fetches a needed repo →
hard-to-diagnose build break (the consumer lacks the repo that would reveal the
missing edge). Extra/dead edges are benign (over-fetch/over-build).

## Read-side vs write-side (why deferral costs nothing structurally)

- **Read-side** — that an incomplete graph can be generated at all — is the hard
  part; it entangles with the optional-flag matrix. Deferred.
- **Write-side** — that an incomplete graph gets *published* as the snapshot — is
  separable and small: gate the re-snapshot action to refuse/warn unless the
  generating configure had every optional flag on. Waiting doesn't make this harder.

## Why deferred

Re-snapshotting `subsystem-{list,edges}` currently triggers a full-from-scratch
CI rebuild — xo-cmake sits at the bottom of the dependency graph, and CI is a
>60min duty cycle — so snapshot updates are intentionally batched/deferred.
Root cause tracked in [02] (decouple-orchestration-data-from-xo-cmake-hash):
decouple these data files from xo-cmake's build-input closure so re-snapshotting
is free (then eager/auto re-snapshot becomes viable and most of this staleness
risk evaporates). 02 is the primary fix and demotes this ticket.

## Acceptance (when picked up)

- Re-snapshot path warns/refuses when its source configure was not all-flags-on.
- (Stretch) top-level umbrella configure warns when the checked-in snapshot
  differs from the freshly generated edges.

## Comments
