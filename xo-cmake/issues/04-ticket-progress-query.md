# 04 — `Progress:` line: let a ticket say how to count its own remaining work

Status: fixed 2026-08-09
Type: feature

## Resolution (2026-08-09)

Implemented as `--progress`, in **`~/proj/xo-sdlc/bin/xo-sdlc.in`** — note the
Files section below originally said `xo-cmake/bin/xo-sdlc.in`, which was wrong:
xo-sdlc lives in its own repo.

```
$ xo-sdlc --tickets --progress
xo-printable2/issues/01-...md   open   [50 left]   [ppsink-migration]
```

Decisions taken:

- **Off by default.** `--tickets` is used casually and must not run shell read
  from a file, nor pay for a tree-wide command per ticket.
- **Convention fixed rather than labelled**: the command prints a count of
  REMAINING items, lower is better, displayed `[N left]`. A per-ticket label
  was considered and dropped as unnecessary complexity for the first cut.
- **Runs from `XO_UMBRELLA2`**, so ticket commands are written tree-relative.
- **Failure is never zero.** A non-zero exit, empty output, or non-numeric
  output all report `[progress?]`. Verified against all three:

```bash
Progress: ... ; false          -> [progress?]
Progress: echo not-a-number    -> [progress?]
Progress: true                 -> [progress?]
```

  A genuine zero reads `[0 left]`, which is visibly distinct. This mattered
  enough to test explicitly: a silent 0 reads as "done", the same defect as
  ctest exiting 0 having run no tests (`.xo-backlog/xo-cmake/issues/03`).

Proven live: converting `DBoolean` took the count from 51 to 50 with no ticket
edit at all.

`xo-sdlc --tickets` shows a ticket's `Status:` and `Milestone:`, which is the
right granularity for most tickets. It has nothing to say about a ticket whose
work is "N of M things converted".

**Raised by RC 2026-08-09**, from using `--tickets` as the entry point to
outstanding work and finding that
`.xo-backlog/xo-printable2/issues/01-aprintable-pretty-ppsink.md` — 55 printers
to convert, one commit each — appears simply as `open`.

## Proposal

A ticket may declare how to count its own remaining work:

```
Progress: grep -rl 'PHASE B STUB' --include=*.hpp xo-*/ | grep -v '/\.build/' | wc -l
```

and `xo-sdlc` runs it, showing the result beside the ticket:

```
xo-printable2/issues/01-aprintable-pretty-ppsink.md   open  [51 left]  [ppsink-migration]
```

Parsed the same way `Status:` and `Milestone:` already are — `grep -m1 '^Progress:'`,
see `xo-cmake/bin/xo-sdlc.in` (`:477`, `:483`).

## Why this shape

**The number cannot go stale.** This is the same argument that made milestones
a query over `Milestone:` lines rather than a hand-kept list, applied one level
down. The evidence is in the motivating ticket: it records the right query, and
also carries a per-subsystem table of counts annotated "as of 2026-08-09;
re-run the grep rather than trusting them". Those counts were wrong within
hours — 55 in the table, 51 in the tree. A written count decays; a command does
not.

**A query already existed and was unreachable.** The information was recorded,
at line 401 of a long ticket. Nothing surfaced it. A `Progress:` line is
mostly about making a query that already exists visible from the tool people
actually run.

**It generalises.** Other tickets are the same shape and currently either lack
a count or keep one in prose:

- `nix-packaging/issues/01` — per-repo nix strategy across many packages
- `subsystem-edges/issues/01,02` — both deferred, both "N places"
- `loc-treemap/issues/01..11` — a numbered pipeline whose progress is countable

## Questions to settle

- **When does it run?** 28 tickets each running a `grep -r` over the tree is
  not free. Options: only under `--tickets --progress`; only when a single
  ticket is displayed; or cache with an explicit refresh. Default should
  probably be OFF, since `--tickets` is used casually.
- **It executes a shell command from a file.** Fine for a personal backlog, and
  xo-sdlc already shells out to `xo-deps`, but it should be a conscious
  decision rather than something that arrives by accident. Worth a note in
  `--help` at least.
- **Working directory.** The command above assumes `XO_UMBRELLA2`. xo-sdlc
  already knows that path (`--u2`), so it can `cd` there — but the convention
  must be stated, or ticket authors will write commands that work only from
  wherever they happened to be.
- **What is displayed?** A bare number is ambiguous — 51 of what, and is
  smaller better? Either the ticket supplies a label
  (`Progress: <label> :: <command>`) or the convention is fixed as "count of
  remaining items, lower is better".
- **Failure.** If the command errors or the ticket is stale, show that rather
  than a zero. A silent 0 reads as "done", which is the failure mode this
  backlog keeps rediscovering — see `.xo-backlog/xo-cmake/issues/03`, where
  ctest exiting 0 with no tests made 26 subsystems look like passes.

**Files:**
- `~/proj/xo-sdlc/bin/xo-sdlc.in` — NOT xo-cmake; `ticket_progress()` beside
  `tickets_list()`
- `.xo-backlog/xo-printable2/issues/01-aprintable-pretty-ppsink.md` — the first
  consumer; its query is at the "The work-list IS a query" section
- `.xo-backlog/CONVENTIONS.md` — where the convention should be documented
  alongside `Status:` and `Milestone:`

**Done when:**
- ~~a ticket carrying a `Progress:` line shows its count in `xo-sdlc`~~ done
- ~~tickets without one are unaffected and cost nothing~~ done — the function
  returns immediately when no `Progress:` line is present, and nothing runs at
  all without `--progress`
- ~~a failing or unparseable command is visibly distinct from zero~~ done
- ~~the convention is recorded in CONVENTIONS.md~~ done — rule 5, beside
  `Status:` (rule 4) and the corrected-diagnoses rule (now 6)

## Notes

Resist letting the ticket carry the *count*. The whole point is that it carries
the *means of counting*. A `Progress:` line whose value is a number, rather
than a command, is the hand-maintained list this backlog already decided
against.
