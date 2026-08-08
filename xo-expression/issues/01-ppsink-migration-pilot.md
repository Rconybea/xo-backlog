# 01 — xo-expression is independent of the facet cluster; pilot or wait?

Status: open (decision needed)
Type: task

`xo-expression` still depends on `xo-indentlog`. It was previously described as
part of the printable2 stack, waiting on the
`IPrintable::pretty(ppindentinfo)` → `PpSink` decision. **That was wrong.**

## Measured (2026-08-08)

xo-expression has no facet-cluster upstream:

```bash
xo-deps --why=xo-expression:xo-printable2     # prints nothing, exit 1
xo-deps --deps-of=xo-expression --format=names -q
#   xo-flatstring xo-indentlog xo-ppsink xo-refcnt xo-reflect
#   xo-reflectutil xo-subsys xo-timeutil
```

`grep -rn "printable2\|xo/facet\|IPrintable" xo-expression` returns nothing.

Of the seven subsystems that still declared an indentlog dep, six are in the
cluster and xo-expression is not:

```bash
for s in $(xo-deps --users-of=xo-indentlog --format=names -q); do
    xo-deps --why=$s:xo-printable2 -q >/dev/null \
        && echo "$s in cluster" || echo "$s INDEPENDENT"
done
```

So nothing about the facet-cluster decision blocks this subsystem. See
`.xo-backlog/CONVENTIONS.md` — the mistaken grouping is the case that prompted
those rules.

## But it is not a quick one

xo-expression has **its own** two-pass pretty protocol, structurally the same
problem as `IPrintable::pretty(ppindentinfo)` and entirely separate from it:

```cpp
// include/xo/expression/GeneralizedExpression.hpp:51
virtual std::uint32_t pretty_print(const ppindentinfo & ppii) const = 0;
```

A pure virtual on the expression hierarchy. It does not stay inside
xo-expression — file counts mentioning `pretty_print`:

| subsystem | files |
|---|---|
| xo-expression | 28 |
| xo-reader | 20 |
| xo-expression2 | 2 |
| xo-reader2 | 2 |

xo-reader implements the same protocol for its parser-state classes, and
inherits indentlog through xo-expression:

```bash
xo-deps --why=xo-reader:xo-indentlog
#   xo-reader -> xo-expression -> xo-indentlog
```

Blast radius, so the scope is bounded rather than guessed:

```bash
xo-deps --users-of=xo-expression --format=names -q
#   xo-expression xo-interpreter xo-jit xo-pyexpression xo-pyjit xo-reader
```

Six subsystems, ~50 files carrying the protocol.

For scale: of 298 files tree-wide mentioning `ppdetail`/`ppindentinfo`,
**233 are inside the printable2 blast radius and 65 are outside it** — of which
xo-expression (28) and xo-reader (24) are 52. Re-measured 2026-08-08; see
`xo-ppsink/issues/02-facility-gaps.md` for the commands. So this stack is
roughly a fifth of the cluster's size and genuinely disjoint from it.

## Everything else it needs is ported

| legacy header | ppsink equivalent |
|---|---|
| `print/quoted.hpp` | `quoted.hpp` / `quoted_ostream.hpp` |
| `print/vector.hpp`, `print/pretty_vector.hpp` | `PrettyVector.hpp` |
| `print/tag.hpp`, `scope.hpp`, `print/ppdetail_atomic.hpp` | `tag.hpp`, `scope.hpp`, (atomic blocks are dead — delete) |

Note `PrettyVector.hpp` still has the old CamelCase name; `PrettyArray.hpp` was
renamed to `pretty_array.hpp` earlier in the migration, so this one is
inconsistent and probably wants the same rename.

`pretty_localenv.hpp` and `pretty_expression.hpp` also include
`<xo/refcnt/pretty_refcnt.hpp>`, the legacy ppdetail forwarder compat header —
so they carry the `ppdetail<rp<T>> -> ppdetail<T>` forwarding problem too.

## The decision

**Pilot it, or wait for the facet-cluster design?**

*For piloting:* it is the same design question at roughly a sixth of the scale,
with no facet machinery in the way, and a bounded, measurable blast radius. A
`pretty_print(ppindentinfo)` → `pretty(PpSink&)` conversion here would surface
the real problems — how a two-pass fit protocol becomes single-pass across a
virtual hierarchy — while they are still cheap to get wrong.

*For waiting:* solving it here risks settling on a shape that does not fit
`IPrintable`, and then either living with two idioms or redoing this one. The
facet cluster is the larger constraint, so arguably it should lead.

Not a coin flip: the sub-question is whether the two protocols are similar
enough that a solution transfers. **Unverified** — nobody has compared
`GeneralizedExpression::pretty_print` against `IPrintable::pretty` side by side.
That comparison is the homework this decision needs, and it is small.

**Files:**
- `xo-expression/include/xo/expression/GeneralizedExpression.hpp:51` — the protocol
- `xo-expression/src/expression/CMakeLists.txt:29` — `xo_dependency(${SELF_LIB} indentlog)`
- `xo-expression/include/xo/expression/{pretty_expression,pretty_localenv,pretty_variable}.hpp`
- `xo-reader/` — 20 files implementing the same protocol

**Done when:**
- the pilot-vs-wait choice is recorded here with its reasoning
- if piloting: xo-expression and xo-reader are off indentlog, verified by
  `grep -l indentlog <build>/**/*.o.d` returning nothing

## Notes

Do the `GeneralizedExpression::pretty_print` vs `IPrintable::pretty` comparison
before choosing. It is the one fact that decides this, and it is cheap.
