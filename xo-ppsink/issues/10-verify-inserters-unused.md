# 10 — prove xo's ostream inserters are unused, by disabling them

Status: open
Type: task
Milestone: ostream-containment

Progress: find xo-*/include -name '*_ostream.hpp' -o -name '*_iostream.hpp' | grep -v '/\.build/' | while read f; do grep -q XO_NO_OSTREAM_INSERTERS "$f" || echo "$f"; done | wc -l

Once every xo `operator<<` lives in a `*_ostream.hpp` / `*_iostream.hpp` bridge
header, add a build-time switch that **compiles the inserter definitions out**,
build the tree with it, and see what breaks. Whatever fails to compile is a
genuine live user; everything else was carrying the dependency for nothing.

RC's call, 2026-08-10. **Sequenced immediately after `ppsink-migration`** — that
milestone deletes xo-indentlog, which owns a large share of the inserters today,
so running this before it would measure a tree that is about to change. It is
part of `ostream-containment` because containment is what makes it possible:
the switch is only a bounded edit once the inserters are all in known files.

## What this answers that containment does not

Containment tells you *where* the inserters are. It does not tell you whether
anything still calls them — and the two are easy to confuse, expensively.

`.xo-backlog/xo-ppsink/issues/02-facility-gaps.md` records the precedent:
`print/printer.hpp` was listed as blocking xo-process on the strength of a grep,
and both hits turned out to be an `#include` and a **commented-out** `using`.
Nothing instantiated it. That cost xo-process a spurious "blocked" label for two
days.

So the check has to discriminate *include* from *use*, which is exactly what
disabling the definition does and what no grep can do.

The payoff if the set comes back empty: the inserters can be **deleted** rather
than merely contained, and `<ostream>` leaves xo's headers altogether.

## Mechanism

Guard the **definitions**, not the headers:

```cpp
/* in each *_ostream.hpp / *_iostream.hpp */
#ifndef XO_NO_OSTREAM_INSERTERS
    inline std::ostream & operator<<(std::ostream & os, const T & x) { ... }
#endif
```

**Not** an `#error` on include. A translation unit that includes a bridge header
without ever writing `os << x` is precisely the case worth distinguishing, and
`#error` would flag it as a user. Guarding the definition lets such a TU keep
compiling and fails only on a real call.

Keep the guard permanently rather than reverting it. It turns a one-off audit
into a repeatable check — a CI job, or one command before anyone adds a new
inserter.

## Running it

```bash
xo-build --sweep -- -DCMAKE_CXX_FLAGS=-DXO_NO_OSTREAM_INSERTERS

# ...then put the tree back.  cmake caches CMAKE_CXX_FLAGS, and a later
# xo-build --configure without -- does NOT clear what it never sets, so
# without this every subsequent build still has the macro defined.
xo-build --sweep --clobber
```

Verified 2026-08-10 on xo-timeutil: a probe macro reached `CMakeCache.txt`,
survived a plain `xo-build --configure`, and was cleared both by `--clobber` and
by naming the variable with an empty value (`-- '-DCMAKE_CXX_FLAGS='`).
`--clobber` is the one to reach for here: this ticket's whole method is to build
the tree in a state you then have to leave completely, and it does not depend on
remembering which variables were touched.

`--sweep` supplies `-q -k --with-utests --with-examples` and forwards `-- ARGS`
to its configure stage only. See `.xo-backlog/CONVENTIONS.md`.

**Corrected twice; both wrong readings kept per `CONVENTIONS.md` rule 6.**

The original command was:

```bash
SUBS=$(xo-build --list | tr ' ' '\n' | grep '^xo-' \
       | while read s; do [ -f "$s/CMakeLists.txt" ] && echo "$s"; done)

xo-build -q -k --configure --with-utests --with-examples --build --install $SUBS \
  -- -DCMAKE_CXX_FLAGS=-DXO_NO_OSTREAM_INSERTERS
xo-build -q -k --utest $SUBS
```

Written by analogy with `xo-reconfigure`, which does take `-- CMAKE_ARGS`, and
never run. At the time `xo-build` had **no such passthrough**: its parser
treated any unrecognised word as a subsystem name, so `--` fell through to the
`*)` case and the run died with a usage message before doing anything —
`xo-build -n --configure xo-timeutil -- -DCMAKE_CXX_FLAGS=-DFOO`, usage, exit 1.

The second version worked around that with `CXXFLAGS=... xo-build --sweep`
preceded by `xo-build --all --realclean`, on the grounds that cmake only reads
the environment variable on a fresh configure. Correct as far as it went, but
obsolete within the hour: `xo-build` gained `-- CMAKE_ARGS` on 2026-08-10, which
needs no realclean because `-D` overwrites the cache entry directly. (`--realclean`
itself was removed later the same day, superseded by `--clobber`, which does the
same thing but runs before the configure rather than after everything.)

The lesson that outlives both: **a command written by analogy and not run is a
guess.** Two revisions of a four-line recipe, in a ticket whose whole subject is
that grep-based reasoning has to be replaced by making the compiler answer.

**`--with-examples` and `--with-utests` are not optional here** — `--sweep`
supplies both — and this ticket is the strongest case for them in the tree.
Measured 2026-08-10, the bridge
headers' includers are concentrated in exactly that code:

| bridge header | includers | of which utest/example |
|---|---|---|
| `tag_ostream.hpp` | 141 | 37 |
| `pretty_ostream.hpp` | 15 | 6 |
| `quoted_ostream.hpp` | 14 | 1 |
| `quantity_iostream.hpp` | 14 | 11 |
| `alloc_ostream.hpp` | 11 | 8 |
| `scaled_unit_iostream.hpp` | 5 | **5** |
| `flatstring_iostream.hpp` | **0** | 0 |

A run without examples would declare `scaled_unit_iostream.hpp` unused when
every one of its includers is example or test code.

Note `flatstring_iostream.hpp` already has **zero includers** — a dead bridge
today, and a preview of what this ticket is looking for.

## Expected result is NOT zero, and that is fine

Some bridges are deliberate paved roads. `xo-webutil/include/xo/webutil/webutil_ostream.hpp`
documents the reasoning in its header comment — PpSink is the intended path
inside xo, ostream is for newcomers and tests — and
`.xo-backlog/xo-alloc/issues/01` plans `alloc_ostream.hpp` on the same basis.

So the deliverable is a **triaged list**, not a pass/fail:

- **intended** — tests, examples, REPL/CLI output, anything whose job is to put
  text on a stream. These justify keeping their bridge.
- **accidental** — library code that should be rendering into a `PpSink` and is
  streaming because the inserter happened to exist. These are conversions, and
  each is a small phase-C-style change: pin the output, convert, re-pin.
- **dead** — a bridge with no users at all. Delete it.

Record the classification here; the count alone will not survive contact with
the next reader.

## Limits worth stating

- **Only xo's own inserters.** A site doing `os << some_std_type` is untouched,
  correctly.
- **Only compiled code.** A use inside a disabled `#if`, an unbuilt subsystem,
  or a template never instantiated will not show up. The two placeholder
  subrepos (xo-equable2, xo-hashable2) and xo-symboltable are already outside
  the 61-subsystem build.
- **A green build proves no *compile-time* use, not that the rendering is
  unwanted.** Something may legitimately want `os << x` in future; that is what
  keeping the guard, rather than deleting the bridges outright, leaves room for.

## Notes

The `Progress:` line counts bridge headers that do not yet carry the guard —
i.e. the mechanical half. It reaching 0 means the switch exists tree-wide, not
that the audit has been run and triaged; this ticket stays `open` until the
list above is written down.

**`xargs grep -L` cannot be used in a `Progress:` line.** `xo-sdlc` is
`set -euo pipefail` (`xo-sdlc:3`) and evaluates the command as written, so any
non-zero stage kills the pipeline and the ticket shows `[progress?]` rather than
a count. `xargs` exits **123** here even though every stage does what it should:

```bash
bash -c "set -euo pipefail; find ... | xargs grep -L PATTERN | wc -l"
#   prints 20, then rc=123
```

Hence the `while read f; do grep -q ... || echo "$f"; done` form above, which is
pipefail-clean. Worth knowing before writing the next `Progress:` line — the
symptom is indistinguishable from a genuinely broken query.

**Files:**
- every `xo-*/include/**/*_ostream.hpp` and `*_iostream.hpp` — 20 of them as of
  2026-08-10, and that number moves with `ostream-containment`
- `.xo-backlog/milestones/ostream-containment.md` — the sweep this completes

**Done when:**
- every bridge header guards its inserters with `XO_NO_OSTREAM_INSERTERS`
- a full build + utest sweep under that flag has been run, `--with-examples`
- every failure is classified intended / accidental / dead, and recorded here
- the dead bridges are deleted
