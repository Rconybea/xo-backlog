# 03 — upgrade the typeseq name table from O(n) to O(1)

Status: open
Type: feature

`typerecd::_by_name(std::string_view)` in
`xo-reflectutil/src/reflectutil/typeseq.cpp` does a linear scan over
`typerecd::s_typerecd_table_`, a `std::vector<typerecd>`. Replace it with an arena-backed
hash map, installed at runtime once one can exist.

Split out of `.xo-backlog/xo-facet/issues/01` on 2026-09-21, which fixed the
correctness bug and deliberately left this.

## Why it was deferred, and why that is still the right call

**The lookup is paid once per (type, module), not per call.**
`typerecd::recd<T>()` memoises the whole `typerecd` in a function-local
static, so a scan happens only on a given module's first ask for a given type. With ~100 types that is a
few thousand string compares across a process's startup.

So do this when something MEASURES it, not on principle. The one number that
would change the calculus is the type count: the table is
`typerecd::table_z()` and the counter `typerecd::id_count()`, and both are
worth looking at before assuming.

```bash
# rows, and ids drawn (the two differ by the internal-linkage types)
.build/xo-reflectutil/utest/utest.reflectutil "[typeseq]" -s | grep -i table
```

Deferring also kept a trap closed. Issue 01 warns that the implementation
pointer must not be a static in a header, or it duplicates per pybind module
exactly as `s_next_id` did and the original bug returns wearing a hat. With no
upgrade to install there is no pointer, and no `typeseq_install_registry()`
sitting unused -- the `arena_streambuf` shape from
`.xo-backlog/xo-arena/issues/05`, which compiled and so looked maintained.

## Design, carried over from issue 01

1. **Two-phase, upgrade-in-place.** Phase (a) is what exists now: a bootstrap
   `std::vector`, O(n), no arena, usable during dynamic initialization. Phase
   (b) installs a `DArenaHashMap` and swings a function pointer.

2. **The pointer lives in `typeseq.cpp`**, never in a header. This is the whole
   reason xo-reflectutil gained a compiled part.

3. **`TypeRegistry` moves from xo-facet to xo-arena**, gaining a
   `DArenaHashMap` for name -> id beside its existing id -> name vector. Its
   includes are `typeseq`, `DArenaVector` and three ppsink headers, all at or
   below arena. `FacetRegistry` includes it and reaches down.

4. **xo-arena acquires appcx machinery.** `AppConfig`/`AppContext` live in
   xo-subsys, which arena already depends on. `ArenaAppcx` becomes the bottom
   of the context chain, displacing `Indentlog2Appcx` -- about 10 tag-list
   sites across 7 files, mostly test mains plus
   `xo-interpreter2/src/skrepl/skreplxx.cpp`.
   `FacetConfig::type_registry_capacity()` moves to arena's config, a better
   home for an arena-backed structure anyway.

5. **Upgrade at `ArenaAppcx` construction:** copy every (name, id) the bootstrap
   already assigned into the hashmap VERBATIM, then swing the pointer.

This step 4 is the bulk of the risk and none of the benefit to the id lookup
itself -- it is there because the hashmap needs an arena and the arena needs a
context. Worth asking, when this is picked up, whether the table could own a
private arena instead and leave the context chain alone.

## Invariants the upgrade must not break

1. **Ids are never reassigned.** The upgrade is additive. A trivial program
   linking libxo_object2 draws 12 ids before `main()`, all already cached in
   per-type statics; reassignment would silently invalidate every one.
2. **The upgrade precedes threads.** Stated as the model. Note the bootstrap
   already takes a `std::mutex` (added by issue 01, because one shared counter
   makes a cross-module race reachable), so the upgrade must decide whether the
   installed implementation still needs it.
3. **Ids stay dense and sequential**, so `TypeRegistry::_id2name`'s
   `DArenaVector` indexing by `seqno()` keeps working. Note an id is NOT a
   table index -- internal-linkage types draw from the counter without adding a
   row, so the two diverge. Pinned by `an-id-is-never-reassigned` in
   `xo-reflectutil/utest/typeseq.test.cpp`; measured 2026-09-22 that returning
   the table index instead leaves every OTHER case in that file green, because
   catch2 runs `ids-are-dense-and-sequential` before any anonymous draw has
   occurred, when the two are still equal.
4. **The table BORROWS its keys today; decide whether the hashmap should.**
   Corrected 2026-09-22 -- the bootstrap no longer copies. `s_typerecd_table_`
   holds `typerecd`, whose `name_` is a `std::string_view`, so a row points at
   whichever module first registered that name
   (`type_name_holder<T>::value` is a per-module header static).

   Sound as it stands, and the soundness is structural rather than
   conventional: `typerecd::_by_name` is PRIVATE, its only production caller is
   `recd<T>()`, and its argument is `type_name<T>()` -- static storage
   duration. The access restriction IS the lifetime contract, which is why
   `typerecd_utaccess` (forward-declared, befriended, defined only in a test)
   is the way to reach it.

   Residual risk is `dlclose`: unloading a module that first registered a name
   would leave dangling views in the table AND in other modules' `recd<T>`
   caches, which memoise the whole `typerecd` including that foreign pointer.
   Latent, since python does not unload extension modules.

   So the hashmap has a choice the original design did not: copy into its
   arena, or keep borrowing. Copying removes the `dlclose` hazard and costs
   arena space; borrowing keeps rows pointer-sized and keeps the contract that
   already exists.
5. **Internal-linkage types are not name-keyed.** A name containing
   `{anonymous}` or `(anonymous namespace)` draws from the counter WITHOUT being
   inserted. Two TUs' anonymous types share a spelling and are different types,
   so keying them together is worse than the bug this all fixes.

## Done when

- `typerecd::_by_name` resolves through the hashmap after the upgrade
- **ids survive the upgrade**: draw an id, install, assert the same number
  comes back. This is issue 01's one unmet done-when item, and the invariant
  whose violation is SILENT -- the bootstrap is not observable any other way
- every invariant above still holds, with the existing `[typeseq]` cases in
  `xo-reflectutil/utest/typeseq.test.cpp` unchanged
- `test_no_module_privately_copies_the_id_source` in `xo-pyobject2/utest` still
  passes -- the new pointer must not be a header static
- `xo-build --sweep` ok in both stages

## Provenance

Split from issue 01 on 2026-09-21, at the point where the correctness fix was
done and the remaining work was a levelization change serving performance
nobody had measured.
