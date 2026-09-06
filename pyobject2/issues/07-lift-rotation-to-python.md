# 07 — v2 gate: lift FacetRegistry rotation into python

Status: deferred
Type: feature
Milestone: pyobject2

v1 is static: python classes are keyed by representation, and every facet is
chosen at compile time inside a binder. That cannot express "given this object,
which facets does it support?" — the question a harness eventually wants, and
the one `FacetRegistry` exists to answer.

Deferred deliberately. Recorded because it gates two other things:

- **`collect()`.** Levelization already forces it out of `AllocFlywheel`:

  ```bash
  grep -n add_subdirectory CMakeLists.txt | grep -E 'xo-(facet|alloc2)\)'
  ```

  `ACollector` is in xo-alloc2, `AllocFlywheel` in xo-facet, so collection must
  be bound from a module at or above xo-alloc2 taking flywheel and collector
  separately — never `fw.collect()`.
- **An erased `Obj` python type**, holding `obj<ATop>` with methods resolved by
  rotation at call time, rather than one class per repr.

The open question is whether v2 is additive or a rewrite of v1's surface. It is
additive if the erased type is a *sibling* of the per-repr classes rather than a
replacement — worth deciding before v1's surface is depended on.
