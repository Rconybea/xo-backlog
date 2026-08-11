# 05 — xo-deps includes the subject in --deps-of / --users-of; make it opt-in

Status: diagnosed
Type: task

`xo-deps --deps-of=X` and `--users-of=X` include **X itself** in their output.
Raised by RC 2026-08-10: that should be **opt-in** (a flag), not the default.

Two separate things came out of looking at it: an API change RC is asking for,
and — independent of that — a place where the implementation does not honour
its own documented contract.

## 1. The inclusion is deliberate and documented

Not an oversight. It is stated twice, in the usage text
(`xo-cmake/bin/xo-deps.in:25-26`):

```
  --deps-of=LIB              Only LIB and what it depends on (upstream)
  --users-of=LIB             Only LIB and what depends on it (downstream)
```

and in the `names` renderer (`:337-339`):

```sh
# every subsystem in the selected cone, one per line.  The focus itself
# is included -- "what depends on Y" reads naturally as including Y.
```

So this ticket is an **API change**, not a bug fix, and the reasoning above
should be answered rather than ignored: "the selected cone" is the right notion
for the graph outputs (svg/dot/html *should* draw the focus node), and
`--format=names` inherited it.

That is the actual seam. `close_over()` sets `keep[start] = 1`
(`xo-deps.in:260`) so the focus is retained, which is correct for rendering; the
`names` format then reuses the same edge-filtered set. One notion of "the cone"
serving two outputs with different needs.

**Suggested shape:** default `--format=names` to excluding the subject, with an
explicit flag (`--with-self`) to restore it; leave the graph formats alone. The
usage text and the `:337` comment both need updating with it — the comment is
the reasoning a future reader will find first.

## 2. Independent bug: an isolated subject vanishes entirely

Whatever is decided above, this is wrong today:

```bash
xo-deps --deps-of=xo-timeutil --format=names -q     # prints NOTHING
xo-deps --deps-of=xo-ppsink   --format=names -q     # xo-ppsink xo-timeutil
```

`xo-timeutil` has no dependencies, so it prints nothing — **not even itself**,
contradicting the documented "the focus itself is included".

Cause: names are derived from surviving *edges*, not from the kept node set
(`xo-deps.in:344`):

```sh
filter_edges | awk '{print $1; print $2}' | sort -u
```

A node with no edges in the cone contributes no names. So the self-entry appears
only when the subject has at least one other dependency — and an empty result
means two different things ("no dependencies" vs "nothing to report") with no
way to tell them apart.

Fixing (1) by emitting the kept node set rather than the edge endpoints would
fix (2) as a side effect, which is an argument for doing it that way.

## 3. What it has already cost

`.xo-backlog/xo-expression2/issues/01-binding-prettifier.md` said xo-reflectutil
is used by **49 subsystems**:

```bash
xo-deps --users-of=xo-reflectutil --format=names -q | wc -l                          # 49
xo-deps --users-of=xo-reflectutil --format=names -q | grep -vx xo-reflectutil | wc -l # 48
```

The real figure is 48; corrected in that ticket. `--format=names` is explicitly
"a query, not an artifact… default to stdout so it pipes" (`:237-238`), so it is
the mode most likely to be fed to `wc -l` or a `for` loop — which is exactly
where an unexpected extra row does damage quietly.

## Corrected diagnosis — the first version of this ticket blamed CMake

**Wrong reading, kept per `CONVENTIONS.md` rule 6.** This ticket originally said
the self-edge originated in the CMake macros, because every `xo_add_*_library`
seeds the target's `xo_deps` property with the target's own name
(`xo-cmake/cmake/xo_macros/xo_cxx.cmake:870,999,1022,1861`) and
`xo_export_cmake_config` has to strip it again (`:1408`, "drop the self-edge
before emitting"). It proposed fixing it there as one of two options.

**That is a different self-edge.** RC's call, and checking it took one command:

```bash
awk 'NF>=2 && $1==$2 {n++} END {print "self-rows:", n+0}' xo-cmake/etc/xo/subsystem-edges
#   self-rows: 0        (out of 201 rows)
```

The edge data is clean. The `xo_deps` property is an internal accumulator whose
only consumer already handles its seed; it never reaches `subsystem-edges`, and
therefore never reaches `xo-deps`. **The problem is isolated to the script.**

Why the wrong version was plausible: both are called a self-edge, both are in
xo-cmake, and the `REMOVE_ITEM` comment describes the symptom in almost the same
words. Two mechanisms with the same name, one of which was already handled.
The check that separates them — grep the actual data file — is trivial and was
not run before writing.

## Suggested approach

Confirm nothing depends on the current behaviour before changing the default:

```bash
grep -rn 'deps-of=\|users-of=' --include=*.sh --include=*.in --include=*.md \
  . 2>/dev/null | grep -v '/\.build/'
```

The loop documented in `CONVENTIONS.md` uses `--users-of` piped into `--why`,
and would gain a spurious `xo-foo -> xo-foo` iteration today; it is unaffected by
the fix. That is one call site, not an audit.

**Files:**
- `xo-cmake/bin/xo-deps.in:260` — `keep[start] = 1` in `close_over()`
- `xo-cmake/bin/xo-deps.in:337-345` — the `names` renderer, its comment, and the
  edge-endpoint derivation behind problem (2)
- `xo-cmake/bin/xo-deps.in:25-26` — usage text stating the current contract
- `.xo-backlog/CONVENTIONS.md` — documents `xo-deps` as *the* way to check
  dependency claims; worth a note once settled

**Not in scope:** `xo_cxx.cmake`'s `xo_deps` seeding, and
`xo-cmake/etc/xo/subsystem-edges`. Both verified correct above.

**Done when:**
- `--format=names` excludes the subject by default, with a flag to include it
- `--deps-of=xo-timeutil` and `--deps-of=xo-ppsink` behave consistently under
  both settings — no more "empty means two different things"
- the usage text and the `:337` comment describe whatever is chosen
- graph formats (svg/dot/html) still draw the focus node
