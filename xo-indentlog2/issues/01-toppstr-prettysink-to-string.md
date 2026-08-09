# 01 — `toppstr()`: render through a PrettySink and return the string

Status: open
Type: feature

xo-indentlog2 has no "render this value with line breaking, give me a string"
entry point. `PrettySink` is the sink that actually breaks lines — ppsink's own
`tostr()` uses a `FlatSink` and never breaks — so anyone wanting broken output
constructs a PrettySink by hand.

**23 files do exactly that.** Measured 2026-08-09:

```bash
grep -rln 'PrettySink' --include=*.cpp --include=*.hpp xo-*/ | grep -v '/\.build/' \
  | while read f; do grep -q 'ArenaConfig' $f && echo "$f"; done | wc -l   # 23
```

**Three of them are production code, not tests** — so this is a missing library
function, not test scaffolding:

```
xo-interpreter/src/interpreter/Schematika.cpp
xo-interpreter2/src/skrepl/skreplxx.cpp
xo-reader/examples/exprreplxx/exprreplxx.cpp
```

## The footgun it would encapsulate

Six of the copies carry the same warning comment, which means six authors hit
or were warned about the same trap:

```bash
grep -rl 'arena name must be unique\|sharing an ArenaConfig name' xo-*/ --include=*.cpp   # 6
```

> NB the arena name must be unique per call: two PrettySinks sharing an
> ArenaConfig name interfere, and the symptom is wrong indentation in whichever
> case runs second.

A warning comment copied into six files is a defect the interface should make
unreachable. The existing copies handle it with a `static int seq` counter,
which is exactly the sort of thing that belongs inside the function once rather
than being rediscovered.

## Shape

Named `toppstr` to match what callers already reach for: `xo-alloc`'s copy is
literally called that, and legacy `toppstr2(ppconfig, x)`
(`xo-indentlog/include/xo/indentlog/print/ppstr.hpp:178`) is what it supersedes.

```cpp
/** render @p x through a PrettySink at soft right margin @p margin **/
template <typename T>
std::string toppstr(const T & x, std::uint32_t margin);
```

Questions to settle:

- **Margin defaulted, or required?** Legacy `toppstr2` takes a whole
  `ppconfig`. A margin-only overload covers most call sites; a
  `PpConfig`-taking overload keeps the rest expressible.
- **Arena sizing.** Every copy hardcodes `64*1024`. Fine as a default, but a
  caller rendering something huge needs a way to raise it.
- **Does it belong beside `tostr`?** ppsink's `tostr` and this differ only in
  which sink they use. Callers will reasonably expect symmetry in naming and
  signature; worth making them read as a pair even though they live in
  different subsystems (ppsink cannot host this — PrettySink is indentlog2's).

## Why now

`.xo-backlog/xo-printable2/issues/01-aprintable-pretty-ppsink.md` phase C needs
to render each converted printer at a margin and compare with the legacy
protocol's rendering at the same margin. Without this, phase C would add a
24th hand-rolled copy — and it would need one per cluster subsystem, so six
more.

But the duplication predates that work and is worth removing on its own.

**Files:**
- `xo-indentlog2/include/xo/indentlog2/print/` — where it lands
- `xo-alloc/utest/ObjectStatistics.test.cpp` — the copy to model it on,
  including the hazard comment worth preserving in the implementation
- the 23 call sites, which can migrate opportunistically rather than all at once

**Done when:**
- `toppstr(x, margin)` exists in xo-indentlog2 and is tested
- the arena-name hazard is unreachable through it
- at least the copies in xo-alloc, xo-object and xo-indentlog2's own utests are
  migrated, so the function is proven against existing expectations

## Notes

Do not fold ppsink's `tostr` into this. They are different renderings — flat
versus broken — and the migration has already been bitten once by treating
"prints the same thing" as "is the same thing" (see
`.xo-backlog/xo-ppsink/issues/02`, on comparing API *shapes* rather than
facility names).
