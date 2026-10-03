# 03 — `DList.cpp` calls `barrier_assign()` without its definition (clang link failure)

Status: fixed 2026-10-03 -- umbrella `ccf65117` (gcc verified; clang CI pending)
Type: bug

## Symptom

GitHub `cmake-docker`, clang job, `build xo-object2` -- linking utest.object2
(e.g. run 37098295886, umbrella `3a793767`; gcc job green):

```bash
gh run view 37098295886 --log-failed | grep 'undefined reference'
# ld: ../src/object2/libxo_object2.so.0.1: undefined reference to
#   `xo::mm::RAllocator<xo::facet::OObject<xo::mm::AAllocator, xo::facet::DVariantPlaceholder> >
#      ::barrier_assign(void*, obj<AGCObject, DVariantPlaceholder>*, obj<AGCObject, DVariantPlaceholder>)'
```

From `9e251d5a` the clang job stopped earlier, at xo-reflect
(`.xo-backlog/xo-reflect/issues/03`, test fixed in `af3fe344`), which masked
this.

## Cause

`RAllocator<O>::barrier_assign()` is declared in
`xo-facet/include/xo/facet/alloc/RAllocator.hpp` and defined out of line in
`xo-alloc2/include/xo/alloc2/alloc/RAllocator_aux.hpp`, which is pulled in by
`xo-alloc2/include/xo/alloc2/Allocator.hpp`. RAllocator_aux.hpp says
"Translation units that want to invoke barrier_assign() must #include
Allocator.hpp". `xo-object2/src/object2/DList.cpp` calls `mm.barrier_assign()`
but did not include it, directly or transitively. Local gcc build, before the
fix:

```bash
cd .build/xo-object2/src/object2/CMakeFiles/xo_object2.dir
for f in DList DArray; do
  echo "$f: aux $(grep -c RAllocator_aux.hpp $f.cpp.o.d)," \
       "$(nm -C $f.cpp.o | grep 'RAllocator<.*>::barrier_assign(void\*' | awk '{print ($1=="U")?"U":$2}' | sort -u)"
done
# DList: aux 0, U     -- references it, cannot instantiate it
# DArray: aux 1, W    -- sees the definition (via DArray.hpp -> Allocator.hpp), emits a weak copy
```

So with gcc the library links only because DArray.cpp's weak copy happens to
satisfy DList.cpp's reference. Under clang it does not -- presumably clang
inlines the call in DArray.cpp and emits no out-of-line copy (UNVERIFIED: the
local clang++, used for the source map, cannot link, so not reproduced
locally).

## Fix

`#include <xo/alloc2/Allocator.hpp>` in DList.cpp, per RAllocator_aux.hpp's
rule. After: `DList: aux 1, W` -- DList.cpp instantiates its own copy, depending
on no other TU.

Other callers: `grep -rn 'barrier_assign(' --include=*.cpp xo-*/ | grep -v '/\.build/'`
-- DList.cpp and DArray.cpp only (DArray.cpp already sees the definition).

## Verified

- gcc: build, ctest 49/49; `nm` as above.
- `xo-build --sweep -j 8`: 73 attempted, 73 ok (build); 47 ok + 26 with no
  tests (utest); `--sweep ok (build and utest)`.
- clang: the CI clang job is the check (expected to get past xo-object2).
