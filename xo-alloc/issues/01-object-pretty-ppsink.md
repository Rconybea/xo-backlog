# 01 — Object::display(std::ostream&) -> Object::pretty(PpSink&)

Status: fixed 2026-08-08
Type: task
Milestone: ppsink-migration

## Resolution (2026-08-08)

`virtual void display(std::ostream&)` is gone from the `xo::gc::Object`
hierarchy, replaced by `virtual void pretty(xo::pp::PpSink&)`. The inserter
moved to an opt-in `xo-alloc/include/xo/alloc/alloc_ostream.hpp`, following
`xo-webutil/include/xo/webutil/webutil_ostream.hpp`.

Done as approach (b), bridged, in four commits-worth of steps, tree green
throughout. The bridge went **one direction only** — `Object::pretty()`
defaulted to `display()`, while `Object::display()` printed directly and never
delegated back. That is what kept the two defaults from recursing for a
subclass overriding neither, and it let unconverted subclasses keep rendering
correctly while their subsystem waited its turn.

### Scope, as built

| subsystem | overrides converted |
|---|---|
| xo-object | 6 (List, Float, String, Integer, Boolean, Primitive) |
| xo-interpreter | 4 (ExpressionBoxed, VsmStackFrame, GlobalEnv, LocalEnv) |
| xo-alloc | 2 (Blob, Forwarding1) + 2 test-local subclasses |

44 files across **five** subrepos: the three above, plus xo-ordinaltree (a
caller of the inherited method) and xo-allocutil (the mirrored fallback base).
Neither of the last two was predicted — see correction 3.

### Rendered output is unchanged, and that is checked

Baselines were captured BEFORE any production change, and still pass unaltered:

- `xo-object/utest/display_baseline.test.cpp` — Integer, Float, String,
  Boolean, List (incl. nested and mixed-type)
- `xo-alloc/utest/display_baseline.test.cpp` — Blob, Forwarding1
- `xo-interpreter/utest/display_baseline.test.cpp` — LocalEnv

Each baseline file was mutation-checked (deliberately break one expectation,
watch it fail, revert) so it cannot be passing vacuously.

`GlobalEnv`, `VsmStackFrame` and `ExpressionBoxed` are NOT pinned — they need
fixtures (a `GlobalSymtab`, a `VsmInstr*`) that the utest has no cheap way to
build. Said plainly in the test file rather than left looking like coverage.

### What the conversion actually buys, also pinned

`xo-alloc/utest/object_pretty.test.cpp` and
`xo-object/utest/object_pretty.test.cpp` assert the thing flat output cannot
show: a nested object participates in the enclosing structure's line breaking.
Before, `gp<Object>` fell through `Prettifier`'s empty primary template to
`operator<<`, arriving as ONE atomic token:

```
<outer
  :b
   <blob :z 64>>      <- could not break at any margin
```

`List` gained real breaking too: element separators are `sink.split(1)`, which
renders as the old single space when flat.

## Three things this ticket did not predict

### 1. The Prettifier must cover the hierarchy, not just gp<Object>

Written first as an exact `Prettifier<gp<Object>>`. The inserter it replaced
took `gp<Object>` and accepted a `gp<Derived>` by **implicit conversion**;
template argument matching grants no such conversion, so every derived type
silently fell back to `operator<<`. Invisible until step 4 deleted that
fallback, then surfaced as `gc_ptr<GlobalEnv>` failing inside `PpSink.hpp`.

Correct form, now in `Object.hpp` with the reasoning beside it:

```cpp
template <typename T>
    requires std::derived_from<T, xo::Object>
struct Prettifier<xo::gp<T>> { ... };
```

No test would have caught this: it is only observable once the fallback is
gone.

### 2. There are TWO mirrored base interfaces

`xo-ordinaltree`'s `RedBlackTree` picks its base from its allocator:

- gc allocator → `xo::Object` → renders `"<Object>"`
- otherwise → `xo::gc::FallbackObjectInterface`
  (`xo-allocutil/include/xo/allocutil/gc_allocator_traits.hpp`) → renders
  nothing

`RedBlackTree` overrides neither — the one under `#ifdef SET_ASIDE` is disabled
— so its `operator<<` called whichever base it inherited, and deleting
`Object::display()` broke the gc path.

**Both bases are now converted, and xo-allocutil took on no dependency.**
`FallbackObjectInterface::pretty(xo::pp::PpSink &) const {}` is a no-op, and a
no-op never touches its parameter, so a reference to a **forward-declared**
`PpSink` suffices — `namespace xo::pp { class PpSink; }` at the top of the
header, no include.

That distinction is load-bearing rather than stylistic. Measured 2026-08-08:

```bash
xo-deps --deps-of=xo-allocutil --format=names -q    # empty, before and after
xo-deps --why=xo-allocutil:xo-ppsink -q             # rc=1, no path
find xo-allocutil/.build -name '*.o.d' | xargs grep -lE 'xo/ppsink/' | wc -l   # 0
xo-deps --users-of=xo-allocutil --format=names -q | wc -l                      # 15
```

xo-allocutil is a foundational leaf with **no dependencies at all** and 15
subsystems above it. Including `PpSink.hpp` would have given it its first
dependency and propagated that across all 15 — to serve a method that does
nothing. Do not "tidy" the forward declaration into an include.

NB the header did not include `<ostream>` or `<iosfwd>` either; `std::ostream`
was arriving transitively through `<memory>`/`<type_traits>`. The explicit
forward declaration is sturdier than what it replaced.

With the mirror restored, `RedBlackTree`'s inserter needs no compile-time
branch: both bases spell it `pretty(PpSink&)`, so it calls that unconditionally.
An earlier fix here used `if constexpr (requires ...)` to detect which base was
present; that complexity existed only while the mirror was broken.

### Why FallbackObjectInterface exists at all

Stated by RC (2026-08-08): **it predates `if constexpr`.** It supplies no-op /
`assert(false)` defaults so call sites can name these members unconditionally,
which was the only option before compile-time branching.

Worth recording because nothing in the code says it, and the class reads as a
deliberate abstract base that a careful reader will preserve on principle —
this ticket's author proposed exactly that, and separately proposed deleting
it, both from misreading its purpose.

It does not follow that the class is simply obsolete: the GC members
(`_shallow_size`, `_shallow_copy`, `_forward_children`) are harder to replace
with `if constexpr` than the rendering one was, and RC's concern there is their
being `virtual`. Left alone.

### 3. The blast radius was callers, not overrides

The 2026-08-08 re-measurement in this ticket ("12 overrides across 3 built
subsystems") is correct **about overrides** and was wrong as a statement about
what the change touches. It was arrived at by grepping three subsystem
directories, which cannot see a caller of an INHERITED method in a fourth.

The audit that works here is the build, not grep: deleting the base method
turns every stale caller into a compile error. `xo-ordinaltree` surfaced that
way, as did five call sites needing the new ostream header —
`xo-object/utest/{Boolean,Integer,List,String}.test.cpp` and
`xo-interpreter/src/interpreter/Schematika.cpp` — where this ticket predicted
only `List.test.cpp`. xo-allocutil surfaced one step further out again, only
once xo-ordinaltree was fixed and rebuilt against a non-gc allocator.

Final radius: 3 subsystems by override count, **5 subrepos** by what actually
had to change.

Generalising, and it is the same lesson as `xo-ppsink/issues/02`: **an
enumeration of definitions is not a measurement of impact.**

## Verified

```bash
SUBS=$(xo-build --list | tr ' ' '\n' | grep '^xo-' \
       | while read s; do [ -f "$s/CMakeLists.txt" ] && echo "$s"; done)
xo-build -q --configure --with-utests --with-examples --build --install $SUBS
for s in $SUBS; do ctest --test-dir $s/.build; done
```

61 subsystems build clean; 34 suites pass, 1 fails — `xo-jit machpipeline.fptr`,
the documented pre-existing failure (`.xo-backlog/xo-jit/issues/01`).

NB when scripting that sweep, do not read the build's status out of `$?` after
a pipe into `grep` — that reports grep's exit, and a failing build reads as
success. It briefly did here.

```bash
nix-build ci.nix -A xo-object --no-out-link    # exit 0
```

That one earns its keep for this ticket specifically: step 4 changed xo-alloc's
public header surface (`operator<<` left `Object.hpp` for `alloc_ostream.hpp`),
and nix is the only check that exercises an installed package config the way a
real consumer would.

## Follow-ons, not filed

- `xo-interpreter/src/interpreter/Schematika.cpp` — the REPL prints results with
  `cout << value << endl` and already holds a `PrettySink` (there is a
  commented-out `pps.pretty(value)` beside it). Rendering results through that
  sink would give the REPL line-broken output. Deliberately not done here: this
  conversion was to leave rendered output unchanged.
- ~~`FallbackObjectInterface::display`~~ — done, see 2 above. `display()` is now
  retired from both mirrored bases.
- `GlobalEnv` / `VsmStackFrame` / `ExpressionBoxed` baselines — see above.

Deferred out of the xo-object ppsink migration (2026-08-07), which was scoped to
the plain indentlog->ppsink swap. This is the follow-on API change, and it is
**not** an xo-object change: `display()` is a virtual declared in xo-alloc.

Same shape as the xo-webutil refactor (`pretty(PpSink&)` + `Prettifier<T>` +
`display_string()` + an opt-in `*_ostream.hpp`), but across a class hierarchy
rather than two standalone classes.

## Scope

Base declaration:

```cpp
/* xo-alloc/include/xo/alloc/Object.hpp:125 */
virtual void display(std::ostream & os) const;      /* non-pure: has a default */
```

**12 live overrides across 3 built subsystems.** Re-measured 2026-08-08; the
earlier count of "17 across 5" (2026-08-07) was an overcount on three separate
grounds, all of them the failure mode this ticket's own NB warns about.

| subsystem | 2026-08-07 | measured 2026-08-08 | |
|---|---|---|---|
| xo-object | 7 | **6** | `Procedure.hpp:33` is commented out |
| xo-interpreter | 4 | **4** | verified `: public Object` / `: public Env` |
| xo-alloc | 4 | **2** | only `Blob.hpp:27`, `Forwarding1.hpp:26` |
| xo-expression2 | 1 | **0** | different hierarchy — see below |
| xo-symboltable | 1 | **(1, dormant)** | not in the build — see below |
| total | 17 | **12** | |

```bash
# declarations, not definitions -- counting both double-counts every override
grep -rn 'display *( *std::ostream.*override' --include=*.hpp xo-*/ \
  | grep -v '/\.build/' | awk -F/ '{print $1}' | sort | uniq -c
```

Plus the inserter at `xo-alloc/src/alloc/Object.cpp:226` (`x->display(os)`).

### Why the three corrections

- **xo-alloc's other `display()` hits are not overrides.** `GcStatistics.hpp`
  (4) and `ObjectStatistics.hpp` (2) declare plain non-virtual `display()` on
  stats classes unrelated to `Object`.
- **xo-expression2 is the *other* cluster, not this one.** `DIfElseExpr.hpp`
  includes `<xo/alloc2/...>`, and the graph agrees:

  ```bash
  xo-deps --why=xo-expression2:xo-alloc       # rc=1, no path
  xo-deps --why=xo-expression2:xo-printable2  # rc=0: xo-expression2 -> xo-printable2
  ```

  So that override belongs to the v2/facet hierarchy gated on
  `APrintable::pretty(ppindentinfo)` — the milestone's 233-file question, and
  not in scope here.
- **xo-object's 7th is a comment**, `Procedure.hpp:33`. Precisely the error
  recorded in `.xo-backlog/xo-ppsink/issues/02` (a commented-out line reads like
  live code to `grep`), repeated here in the ticket that warns about it.

### xo-symboltable is dormant, not abandoned

`Symbol : public Object` (`xo-symboltable/include/xo/symboltable/Symbol.hpp:19`)
is a real override, but the subsystem is not built:

```bash
grep -n symboltable CMakeLists.txt      # 145: #add_subdirectory(xo-symboltable)
xo-build --list | tr ' ' '\n' | grep -c '^xo-'   # 63; symboltable absent
grep -rn 'xo/symboltable/' --include=*.hpp --include=*.cpp xo-*/ \
  | grep -v '/\.build/' | grep -v '^xo-symboltable/'   # no consumers
```

**Stated by RC (2026-08-08): dormant by intent — the code is not abandoned, and
would naturally revive during xo-interpreter work.** So it is out of scope here
(nothing compiles it, so it cannot break the build), but it is *not* dead code
to delete, and whoever revives it inherits the conversion: if this ticket lands
first, `Symbol` comes back needing `pretty(PpSink&)`, not `display()`.

`xo-deps --why=xo-symboltable:...` returns **rc=2 ("no such subsystem")**, which
is correct rather than a registration bug — it genuinely is not a subsystem of
this build. Per `CONVENTIONS.md`, do not read that rc=2 as "not a dependency".

Careful with `grep symboltable` over CMake files: it also matches
`xo-expression2-facet-symboltable` and two `facetimpl-symboltable-*` targets,
which are the v2 stack's own facets and unrelated to `xo-symboltable/`.

`display()` is virtual, so its signature cannot change piecemeal -- every
override must move together, or a bridge must exist during the transition.

**NB there are 12 other, unrelated `display(std::ostream&)` hierarchies in the
tree** (xo-kalmanfilter, xo-reactor, xo-refcnt's `Displayable`, xo-expression,
xo-jit, xo-websock, ...). Do not let a tree-wide grep pull them in; this ticket
is only about the one rooted at `xo::gc::Object`.

## Approach

Two candidates were weighed when this was deferred:

- **(a) flag-day.** Change the base and all 17 overrides in one commit. Fewer
  moving parts, no throwaway bridge -- but no green intermediate state, and one
  commit spanning 5 subsystems.
- **(b) bridged (recommended).** Add `virtual void pretty(PpSink&) const` to
  `Object` with a default, convert subclasses subsystem by subsystem, then
  delete `display()`.

(b) keeps the tree building throughout, which is how every other step of this
migration has gone. It needs one piece of care: if both `display()` and
`pretty()` are virtual and each defaults to the other, a class overriding
*neither* recurses forever. The base must break that cycle -- e.g. `Object`'s own
`display()` default prints directly rather than delegating.

Order for (b): xo-alloc (base + its own 2) -> xo-object (6) -> xo-interpreter (4)
-> delete `display()` and the bridge. Three subsystems, and they are already
consecutive in the dependency order (`xo-interpreter -> xo-object -> xo-alloc`),
so each step's consumers are the next step's subject.

Neither xo-expression2 nor xo-symboltable appears in that order any more: the
first is the v2 cluster's problem, the second is not built.

## Watch out for

- **ADL, not just includes.** In xo-object's public headers a *function-local*
  `using xo::pp::xtag;` was still ambiguous, because the argument type
  (`gp<Object>`) lives in namespace `xo`, so ADL added legacy `xo::xtag` to the
  candidate set. A using-declaration cannot suppress ADL at any scope -- only a
  qualified call can. Expect to qualify inside these headers.
- **`Object.hpp` must stay cheap to include.** It is at the root of the object
  model. `PpSink.hpp` is deliberately ostream-free, so it is a lighter include
  than `<ostream>`; prefer it over pulling in the whole printing stack.
- Consumers of `xo_object` that would newly need ppsink: xo-gc, xo-imgui,
  xo-interpreter, xo-object2, xo-ordinaltree, xo-procedure2, xo-type. (xo-object
  already propagates ppsink publicly as of the migration, so this should be a
  no-op.)

**Files:**
- `xo-alloc/include/xo/alloc/Object.hpp:125` — the base declaration
- `xo-alloc/src/alloc/Object.cpp:226` — the inserter
- the 12 live overrides tabulated above (plus xo-symboltable's, if revived)

**Done when:**
- rendering goes through `pretty(PpSink&)`; `display(std::ostream&)` is gone from
  the `xo::gc::Object` hierarchy
- an opt-in `*_ostream.hpp` provides `os << obj` for callers outside xo
  (see `xo-webutil/include/xo/webutil/webutil_ostream.hpp` for the pattern)
- the tree builds and tests pass at every commit, not just at the end
