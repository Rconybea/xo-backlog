# 09 — filling the root set truncates silently, then segfaults

Status: open
Type: bug
Milestone: pyobject2

Holding more live handles than a flywheel's strong set has slots does not
report anything. `add_strong_ref` hands back a null slot pointer, the handle
stores it, and the first read through that handle dereferences null.

Reproduce from python (the default strong set holds 256 `obj<ATop>`):

```bash
.build/xo-python -c '
import xo.facet as f, xo.object2 as o
fcx = f.configure_all(); fw = f.AllocFlywheel.make_default_app(fcx)
keep = [o.Float.make(fw, float(i)) for i in range(400)]
print("held", len(keep), "count", fw.strong_root_count())
print(keep[-1].value())
'
```

Observed 2026-09-13: `held 400 count 256`, then `Segmentation fault`. Note the
two separate failures — the count says 256 while python holds 400 handles, so
144 of them are silently rootless *before* anything crashes.

The boundary is exact and worth seeing: 256 handles is fine, 257 does not crash
but leaves `strong_root_count() == 256`.

## Pre-existing, not introduced by the free lists

Measured, not assumed: with `02`'s change stashed, the same script segfaults at
the same point.

```bash
git stash push -- xo-facet xo-pyfacet
cmake --build .build --target xo_pyfacet xo_pyobject2 -j8
# ... run the script above: same crash
git stash pop
```

`02` fixed the *reachable* case — a REPL loop that creates and drops handles no
longer exhausts the set, because slots come back. This is the other case:
genuinely more live objects than slots. A free list cannot help, since none of
them are free.

## Where it goes wrong

```bash
grep -n '_add_ref' -A 20 xo-facet/include/xo/facet/handlestore/DHandleStore.hpp
```

`DArenaVector::push_back` returns `nullptr` when the vector is at capacity (a
`DArenaVector` fixes capacity at construction). `_add_ref` returns that pointer
unexamined, and `DObjectHandle::_native()` reads through it:

```bash
grep -n '_native' -A 3 xo-facet/include/xo/facet/ObjectHandle.hpp
```

## The decision this needs

Three options, and the choice is not obvious — which is why `02` did not settle
it in passing:

1. **Throw.** `make_strong_ref` raises when the set is full. Cheapest, and
   turns UB into a diagnosable error naming the flywheel. But it puts an
   exception on an allocation path that a collector may later want to be
   `noexcept`.
2. **Grow the root set.** A `DArenaVector` cannot; this means a different
   structure, or a chain of arenas. Largest change, and it interacts with
   whatever the v2 collector wants to scan.
3. **Report, do not raise.** `make_strong_ref` returns an empty handle and the
   caller checks. Fits the existing `AllocError`/`last_error()` idiom, but every
   binding site then has to check, and a missed check is the same crash.

(1) looks right for now — the set is configurable, so a caller that hits the
limit has a knob — but it should be decided rather than defaulted into.

## Done when

- the script above raises, or returns a handle the caller can test, instead of
  crashing
- a C++ case beside the `[freelist]` cases in `xo-facet/utest/objectmodel.test.cpp`
  fills a deliberately small root set and asserts the chosen behaviour
- `handle-loop-reuses-slots` still passes: it detects capacity by filling the
  set until `_impl_handle()` comes back null, so a change here changes how it
  must probe

## Provenance

Found 2026-09-13 while verifying `02`. Not fixed there: `02`'s subject is
release and reuse, and this needs a design decision of its own.
