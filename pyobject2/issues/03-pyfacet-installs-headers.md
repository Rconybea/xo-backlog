# 03 — xo-pyfacet must install headers, and ship bind_top

Status: open
Type: feature
Milestone: pyobject2

Binder templates are consumed by the python modules above them, so the modules
that own them have to install headers rather than only building a `.so`.
`xo-pyfacet` builds only a `.so` today.

This is not new ground for the tier: `xo-pyutil` already installs headers and
five `xo-py*` modules consume them, `xo-pyfacet` included. Copy its pattern.

```bash
find xo-py*/include -type f | grep -v README         # xo-pyutil's two headers
ls ~/local/include/xo/pyutil/                        # installed
grep -n xo_install_library4 xo-pyutil/CMakeLists.txt # the install rule
```

(`xo-pyreflect/include/` holds only a README, which is what made an earlier
reading conclude no module installed headers.)

## Shape

- `xo-pyfacet/include/xo/pyfacet/bind_top.hpp` — binder for the `ATop` members
  worth exposing (`typeseq`, and `__repr__` fallback).
- A handle alias for pybind classes to hold, over `DObjectHandle<AFacet,DRepr>`.
- `xo-pyfacet` installs its include dir and gains a `Config.cmake` that a
  consumer can `find_package`, with the generated `find_dependency` block
  (`.xo-backlog/generated-find-dependency`).

**Done when:**
- `xo-pyprintable2` can `#include <xo/pyfacet/bind_top.hpp>` through
  `find_package(xo_pyfacet)` alone, with no relative include path
