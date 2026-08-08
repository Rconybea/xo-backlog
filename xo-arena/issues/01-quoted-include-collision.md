# 01 — Quoted `#include "foo.hpp"` from utest/ can silently bind a foreign subsystem's header

Status: partly fixed 2026-08-08
Type: bug

`xo-arena/utest/DArena.test.cpp` and `DCircularBuffer.test.cpp` both said

```cpp
#include "print.hpp"
```

and both were opening **`/home/roland/local/include/xo/randomgen/print.hpp`** —
a different subsystem's header — rather than `xo/arena/print.hpp`.

## Why

A quoted include searches the *including file's own directory* first, then falls
back to the ordinary include path. There is no `xo-arena/utest/print.hpp`, so the
fallback ran, and `<...>/xo/randomgen` sits earlier on the path than
`xo-arena/include/xo/arena`. The sibling includes in the same files
(`"DArena.hpp"`, `"DCircularBuffer.hpp"`) resolved correctly — `print.hpp` was
simply the one basename that also exists in another subsystem.

**This is not a compile error.** The wrong header is a perfectly valid header; it
just doesn't declare what the file needs. The failure surfaces far away, as a
missing overload deep inside a template instantiation:

```
PpSink.hpp:54: error: no match for 'operator<<'
  (operand types are 'std::ostream' and 'const xo::mm::AllocError')
```

`ins.os() << x` looks broken, and `xo/arena/print.hpp` *does* define exactly that
inserter — it was never included. An hour can go into ADL and two-phase lookup
before checking which file the preprocessor actually opened.

## Diagnosis in one step

The `.o.d` files record every header opened, so ask the build rather than
reasoning about search order:

```bash
find xo-arena/.build -name 'DArena.test.cpp.o.d' | xargs grep -o '[^ ]*print\.hpp'
# => /home/roland/local/include/xo/randomgen/print.hpp     <-- wrong subsystem
```

Do this **before** analysing any "impossible" missing-overload error.

## Scope — measured 2026-08-08

31 header basenames exist in two or more `xo-*/include/xo/*/` directories, and 30
of those are used somewhere as quoted includes. Most are harmless: a file in
`xo-object/include/xo/object/` including `"Object.hpp"` finds its own.

The exposed set is *quoted includes from a directory that does not itself contain
the named file* — i.e. `utest/` and `example/` reaching into their own
subsystem's `include/` tree. Only three exist tree-wide:

| Site | Basename also in | Status |
|---|---|---|
| `xo-arena/utest/DArena.test.cpp` | xo-randomgen | **was resolving wrong** — fixed |
| `xo-arena/utest/DCircularBuffer.test.cpp` | xo-randomgen | **was resolving wrong** — fixed |
| `xo-arena/utest/span.test.cpp` → `"span.hpp"` | xo-tokenizer | resolved correctly; fixed anyway |
| `xo-indentlog/utest/log_streambuf.test.cpp` → `"scope.hpp"` | xo-ppsink | resolves correctly today; **left alone** — xo-indentlog is being retired |

All fixed sites now use the unambiguous angle-bracket form with a comment saying
why.

Reproduce the scan with `scan.sh` logic:

```bash
find xo-*/utest xo-*/example -name '*.cpp' -o -name '*.hpp' | grep -v '\.build' | while read f; do
  d=$(dirname "$f")
  grep -oE '#include "[A-Za-z0-9_]+\.hpp"' "$f" | sed 's/#include "//;s/"//' | while read h; do
    [ -f "$d/$h" ] && continue
    n=$(find xo-*/include/xo -mindepth 2 -maxdepth 2 -name "$h" | wc -l)
    [ "$n" -gt 1 ] && echo "AMBIGUOUS($n): $f -> \"$h\""
  done
done | sort -u
```

## Why it appeared now

`xo-arena` migrated to ppsink in `21efdca1`. Before that, the needed `operator<<`
arrived through legacy indentlog regardless of which `print.hpp` was opened, so
the wrong binding cost nothing. Removing indentlog made `xo/arena/print.hpp` the
only supplier — and it was never being included. Same shape as the
`-lindentlog` case in `generated-find-dependency/issues/02`: the migration did
not introduce the bug, it removed what was masking it.

**Files:**
- `xo-arena/utest/{DArena,DCircularBuffer,span}.test.cpp` — fixed
- `xo-indentlog/utest/log_streambuf.test.cpp` — latent, deliberately untouched

**Done when:**
- no quoted include in a `utest/`/`example/` directory can bind outside its own
  subsystem
- ideally, a lint or build check enforces it, since the failure mode is silent

## Notes

Prefer `#include <xo/<subsystem>/foo.hpp>` everywhere except for a file that
genuinely sits beside the one it includes. The quoted form buys nothing here and
costs a silent, action-at-a-distance misbinding whenever a basename is shared.
