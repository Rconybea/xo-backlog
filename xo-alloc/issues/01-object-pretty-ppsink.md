# 01 — Object::display(std::ostream&) -> Object::pretty(PpSink&)

Status: open
Type: task
Milestone: ppsink-migration

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
