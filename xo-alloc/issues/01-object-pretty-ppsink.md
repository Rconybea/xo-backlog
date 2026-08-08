# 01 — Object::display(std::ostream&) -> Object::pretty(PpSink&)

Status: open
Type: task

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

**17 overrides across 5 subsystems** (measured 2026-08-07):

| subsystem | overrides |
|---|---|
| xo-object | 7 |
| xo-alloc | 4 |
| xo-interpreter | 4 |
| xo-expression2 | 1 |
| xo-symboltable | 1 |

Plus the inserter at `xo-alloc/src/alloc/Object.cpp:226` (`x->display(os)`).

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

Order for (b): xo-alloc (base + its own 4) -> xo-object (7) -> xo-interpreter (4)
-> xo-expression2, xo-symboltable (1 each) -> delete `display()` and the bridge.

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
- the 17 overrides listed above

**Done when:**
- rendering goes through `pretty(PpSink&)`; `display(std::ostream&)` is gone from
  the `xo::gc::Object` hierarchy
- an opt-in `*_ostream.hpp` provides `os << obj` for callers outside xo
  (see `xo-webutil/include/xo/webutil/webutil_ostream.hpp` for the pattern)
- the tree builds and tests pass at every commit, not just at the end
