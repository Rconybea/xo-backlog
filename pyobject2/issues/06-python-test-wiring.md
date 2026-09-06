# 06 — python tests for the xo-py* tier

Status: open
Type: feature
Milestone: pyobject2

No `xo-py*` module has tests. The only pytest wiring in the tree is xo-cmake's:

```bash
find . -name 'test_*.py' -not -path './*/.build/*' | grep -v '^./xo-cmake'   # empty
sed -n '1,20p' xo-cmake/utest/CMakeLists.txt                                 # the add_test pattern
```

Most of this milestone's logic does not need python to be tested — handle
narrowing, recovery and root release are all c++ and belong in
`xo-facet/utest` (tickets 01, 02). What only python can assert is the binding
itself, and the lifetime coupling:

- a dropped python reference drops the strong root (`del o` lowers
  `strong_root_count()`)
- an object survives and still renders after an intervening allocation
- a repr missing a facet has no method for it

**Shape:** `examples/*.py` per module, following `xo-pyfacet/examples`, plus one
pytest module wired through ctest so `xo-build --sweep` runs it.

Note `--with-examples` already exists precisely because examples catch breakage
utests miss (`.xo-backlog/CONVENTIONS.md`); an example that imports the module
and renders one object is cheap coverage of the whole stack.

**Done when:**
- `xo-build --sweep` reports the python suite among its `ok` totals, not as
  `no-tests`
