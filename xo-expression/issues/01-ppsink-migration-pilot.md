# 01 — xo-expression/xo-reader: pilot the ppindentinfo -> PpSink conversion here

Status: open (decided 2026-08-08: pilot here; implementation not started)
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

## Decision: pilot here (2026-08-08)

Chosen deliberately so the newer facet-based stack benefits from what the older
one teaches. The comparison below was the homework; it supports the choice.

### The two protocols differ on exactly one axis

```cpp
// older stack -- xo-expression/include/xo/expression/GeneralizedExpression.hpp:51
virtual std::uint32_t pretty_print(const ppindentinfo & ppii) const = 0;

// facet stack -- xo-printable2/include/xo/printable2/detail/APrintable.hpp:51
virtual bool pretty(Copaque data, const ppindentinfo & ppii) const = 0;
```

Same `ppindentinfo` two-pass protocol, same `ppstate` underneath, same
`pretty_struct` in the bodies. The only difference is the **receiver**: the
older one is virtual on the object, the facet one is stateless with the object
passed as `Copaque data`.

That difference is **orthogonal** to the `ppindentinfo` → `PpSink` change. So
whatever shape works here transfers directly:

```cpp
virtual void pretty(xo::pp::PpSink & sink) const = 0;               // older
virtual void pretty(Copaque data, xo::pp::PpSink & sink) const = 0; // facet
```

### The return value is vestigial — it becomes `void`

Two independent pieces of evidence:

1. **Nothing consumes it but the legacy two-pass driver.** Every call site is a
   `ppdetail<T>::print_pretty` specialization forwarding
   `return x.pretty_print(ppii);` (`pretty_expression.hpp:18,26`,
   `pretty_localenv.hpp:14,21,29`, `pretty_variable.hpp:16,23`,
   `pretty_envframestack.hpp:16,23`, `pretty_exprstatestack.hpp:16`). In the
   legacy model the `bool` means "did it fit" during the `upto()` pass. ppsink is
   single-pass and has no such driver — `Prettifier<T>::print` returns `void`.
2. **The two stacks already disagree on its type.** xo-expression declares
   `std::uint32_t`, xo-reader declares `bool`
   (`exprstatestack.hpp:54`, `envframestack.hpp:63`), and the implementations
   `return ppii.pps()->pretty_struct(...)`, which returns `bool`. The `uint32_t`
   is an implicit widening of a bool nobody reads.

### Two-thirds of the work is already mechanical

Most implementations are one-liners delegating to `pretty_struct`, and **ppsink
already has `pretty_struct` + `field()`**. So those convert by substitution:

```cpp
// from
return ppii.pps()->pretty_struct(ppii, "Apply", refrtag("fn", fn_), refrtag("argv", argv_));
// to
sink.pretty_struct("Apply", field("fn", fn_), field("argv", argv_));
```

Measured 2026-08-08 (file counts; overlap resolved):

| | delegate to `pretty_struct` | hand-written raw `upto()` |
|---|---|---|
| xo-expression | 9 | **4** — GeneralizedExpression, GlobalSymtab, LocalSymtab, Sequence |
| xo-reader | 6 | **4** — envframestack, exprstatestack, pretty_parserstatemachine, progress_xs |

`Apply.cpp` appears in both columns only because its raw-`upto()` version sits
in an `#ifdef OBSOLETE` block; `Lambda.cpp` has one too. Both are dead — delete
rather than convert, as with the `ppdetail_atomic` blocks.

**The 8 hand-written files are the actual design work**, and they are precisely
the transferable lesson: they are where a two-pass fit protocol has to become
single-pass. Everything else is substitution.

## Design settled 2026-08-08

Grilled out in full; the decisions below are fixed. Implementation not started.

1. **The protocol becomes** `virtual void pretty(xo::pp::PpSink & sink) const = 0;`
   — renamed from `pretty_print`, return dropped (see above for why it has no
   consumer). `Prettifier<GeneralizedExpression>` forwards to it, per the
   `TypeDescr_pp.hpp` precedent.
2. **No legacy compat header.** Measured: nothing outside xo-expression/xo-reader
   prints an expression through `ppdetail`. xo-jit's only `ppdetail` hit is a
   dead `#ifndef ppdetail_atomic` block (`activation_record.hpp:83`);
   xo-interpreter and xo-jit print only *scalars* off expressions
   (`expr->name()`, `expr->function_address()`). So unlike xo-reflect, no
   `_ppdetail.hpp` residue is needed.
3. **Dynamic-arity fields need new ppsink API** — landing first, separately:
   `.xo-backlog/xo-ppsink/issues/06-dynamic-arity-struct-builder.md`.
4. **The forced-break policy is preserved**, as a `force_break` flag on that
   builder. Behaviour-preserving migration first; layout changes, if wanted,
   in a separate commit where the diff shows only the layout.
5. **`LocalSymtab` keeps `this`.** The fit branch measured `argv` only while the
   print branch emitted `this` + `argv`, so the single-line form could overflow
   the margin — the divergence is a bug **in the fit branch**. Single-pass makes
   this class of bug structurally impossible; there may be more instances among
   the facet stack's 233 files, unchecked.
6. **Scope is xo-expression + xo-reader only.** xo-interpreter and xo-jit
   currently freeload on xo-expression's PUBLIC indentlog propagation, so each
   gets a temporary explicit `xo_dependency(${SELF_LIB} indentlog)`, retired in
   a prompt follow-up. Deliberate: smaller review surface.
   xo-pyexpression/xo-pyjit need nothing (zero indentlog includes).
7. **Verification:** capture a rendered baseline *before* converting, review
   every diff deliberately after, then pin the risk-carrying shapes as new unit
   tests — dynamic arity, forced break at a margin wide enough to fit, optional
   fields present *and* absent, the `LocalSymtab` change, and one plain
   `pretty_struct` type as a control. Nothing pins this output today: xo-expression
   has one utest (`type_unifier`), xo-reader two, none asserting on rendering.
   Margin-sensitive cases need a test-only `xo_indentlog2` dep, following
   `xo-alloc/utest`.

### The 7 hand-written files, by what they actually need

Re-read individually 2026-08-08; three are not hard at all:

| file | needs |
|---|---|
| `GlobalSymtab` | `pretty_struct` — mechanical |
| `progress_xs` | `field(name, val, present)` — its `if (lhs_)` / `if (op_type_ != invalid)` guards *are* the present flag |
| `pretty_parserstatemachine` | a plain `Prettifier<parserstatemachine>` — it is a free `ppdetail<T>`, not a member, so no virtual is involved |
| `LocalSymtab` | `pretty_struct`, keeping `this` (decision 5) |
| `Sequence` | the builder |
| `exprstatestack`, `envframestack` | the builder + `force_break` |

`GeneralizedExpression::pretty_print` is **not** in this set — it sits inside
`#ifdef SUPERSEDED`, so the base is a pure virtual with no fallback. The
`#ifdef OBSOLETE` blocks in `Apply.cpp` and `Lambda.cpp` are likewise dead:
delete rather than convert.

### Commit sequence

1. `xo-ppsink: struct_open builder for dynamic-arity fields [FEATURE]` — issue 06
2. `xo-interpreter/xo-jit: declare own xo-indentlog dep [BUILD]` — harmless
   while still inherited, so it can land first
3. `xo-expression/xo-reader: use xo-ppsink !xo-indentlog [REFACTOR]` — atomic;
   the virtual's signature change means these cannot be split
4. tests + subsystem-edges + nix + `Config.cmake.in`
5. follow-up: retire the xo-interpreter/xo-jit stray deps

**Not yet checked:** the ~15 mechanical `pretty_struct` conversions have not been
read individually. They are one-liners of a known shape, but `LocalSymtab` shows
this file set can hide a fit/print asymmetry — surface any others rather than
silently picking a branch.

**Files:**
- `xo-expression/include/xo/expression/GeneralizedExpression.hpp:51` — the protocol
- `xo-expression/src/expression/CMakeLists.txt:29` — `xo_dependency(${SELF_LIB} indentlog)`
- `xo-expression/include/xo/expression/{pretty_expression,pretty_localenv,pretty_variable}.hpp`
- `xo-reader/` — 20 files implementing the same protocol

**Done when:**
- if piloting: xo-expression and xo-reader are off indentlog, verified by
  `grep -l indentlog <build>/**/*.o.d` returning nothing

## Notes

When converting the 8 hand-written files, record *why* each two-pass body maps
to its single-pass form. That reasoning -- not the diff -- is what the facet
stack needs, and it is the reason this subsystem goes first.
