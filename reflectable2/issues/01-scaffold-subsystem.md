# 01 — scaffold xo-reflectable2

Status: open
Type: task
Milestone: reflectable2

Stand up the subsystem, with no methods, so later tickets have a home and the
levelization is settled before any code depends on it.

`xo-equable2` and `xo-hashable2` were stood up this way — scaffolded facet
subsystems that build with no methods and no tests — so the pattern exists:

```bash
grep -n '^xo-equable2$\|^xo-hashable2$' xo-cmake/etc/xo/subsystem-list
ls xo-equable2 xo-equable2/include/xo/equable2/detail
```

## Shape

1. New subsystem `xo-reflectable2`, entered in `xo-cmake/etc/xo/subsystem-list`
   **above `xo-reflect` and below `xo-stringtable2`**, so the types that opt in
   later can implement the facet in their own subsystems:

   ```bash
   grep -n '^xo-reflect$\|^xo-stringtable2$\|^xo-object2$' xo-cmake/etc/xo/subsystem-list
   ```

2. Dependencies: `xo-reflect` and `xo-facet`. Confirm the edge is legal and
   creates no cycle:

   ```bash
   xo-deps --why=xo-reflect:xo-facet -q; echo "exit=$?"   # expect 1 (no path) before
   ```

3. The facet quartet under `include/xo/reflectable2/detail/`, with no methods
   yet — `03` adds the method. Mirror an existing scaffold rather than
   inventing the file set:

   ```bash
   ls xo-equable2/include/xo/equable2/detail/
   # AEquable.hpp  IEquable_Any.hpp  IEquable_Xfer.hpp  REquable.hpp
   ```

   so: `AReflectable.hpp`, `IReflectable_Any.hpp`, `IReflectable_Xfer.hpp`,
   `RReflectable.hpp`. Re-run that `ls` rather than trusting this list — it is
   the scaffold as of 2026-09-12.

**Done when:**
- `xo-build --sweep` is green and the subsystem is swept, not merely listed.
  Note the installed list, not the source list, decides what `--all` covers:

  ```bash
  comm -13 <(grep '^xo-' ~/local/share/etc/xo/subsystem-list | sort) \
           <(grep '^xo-' xo-cmake/etc/xo/subsystem-list | sort)   # expect empty
  ```

  so xo-cmake must be installed before the sweep will cover the new entry.
  See the sweep section of `CONVENTIONS.md`.
- the sweep's `attempted` total rises by one; that is the confirmation, and a
  total that does NOT move means the entry did not reach the installed list
