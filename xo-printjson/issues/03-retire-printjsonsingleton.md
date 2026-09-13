# 03 — retire PrintJsonSingleton in favour of PrintJsonAppcx

Status: open
Type: refactor

`PrintJsonAppcx` (added 2026-09-13) gives each application context its own
`PrintJson`, so json printer registration stops being a process-wide side
effect. Finish the job: give every subsystem that reaches for
`PrintJsonSingleton` an Appcx of its own, migrate them one at a time, then
delete the singleton.

## Why this is worth doing

Printer registration is POLICY -- which types render how -- and policy belongs
to an application, not to a process. Type IDENTITY is genuinely global, and
correctly stays so: `ReflectAppcx` holds `TypeDescrTable::instance()`,
`EstablishTypeDescr::establish<T>()` interns into a function-local static, and
`FacetRegistry`/`TypeRegistry` are untouched. The Appcx split separates the two,
which the singleton design conflated.

A second gain, already banked: the `Deps` template constructor makes
levelization COMPILER-CHECKED. `Object2Appcx` cannot be built without
`deps.cx<S_printjson_tag>()`, so the xo-printjson levelling (`58ec6b70`) is now
enforced where a context is assembled, not merely by the subsystem-list.

## The hazard this ordering avoids

Today `PrintJsonAppcx` ADOPTS the singleton rather than constructing its own:

```bash
grep -n 'print_json_ =' xo-printjson/src/printjson/PrintJsonAppcx.cpp
#   this->print_json_ = PrintJsonSingleton::instance();
```

That is what keeps the tree working mid-migration: appcx-registered and
singleton-registered printers land in ONE map, so an unmigrated consumer still
sees object2's printers and vice versa.

**The moment `print_json_` is constructed fresh, that stops being true.** A
printer registered through the singleton is then invisible to a context, and
the failure is a MISSING printer -- json that is subtly wrong -- not a link
error. So the fresh-construction switch is the LAST step, after the count below
reaches zero, not a step along the way.

## Progress

Carry the means of counting, not the count. Grep the SYMBOL, not the header:

```bash
grep -rl 'PrintJsonSingleton' --include=*.cpp --include=*.hpp xo-*/ \
  | grep -v '/\.build/' | sed 's|/.*||' | sort -u
```

As of 2026-09-13 that is 8 subsystems: xo-kalmanfilter, xo-object2,
xo-printjson, xo-pyprintjson, xo-pyprocess, xo-pyreactor, xo-pywebsock,
xo-websock.

### Why the symbol, not the header

RC's proposed criterion -- a subsystem no longer pulls `PrintJsonSingleton.hpp`
-- is right in spirit and leaks in practice. `xo-websock` uses the symbol
without including the header and so does not appear in a header grep:

```bash
grep -rl 'PrintJsonSingleton.hpp' --include=*.cpp --include=*.hpp xo-*/ | grep -v '/\.build/' | sed 's|/.*||' | sort -u   # 7
grep -rl 'PrintJsonSingleton'     --include=*.cpp --include=*.hpp xo-*/ | grep -v '/\.build/' | sed 's|/.*||' | sort -u   # 8
```

The header test would call xo-websock done while it still uses the singleton.
`PrintJson.hpp` no longer declares it (extracted to its own header by
`b63f8fbb`), so the two greps agree for everything else -- but a criterion that
depends on that staying true is one regeneration away from being wrong.

## Per-subsystem notes

- **xo-object2** — done; `Object2Appcx` registers via
  `pjson_appcx.print_json()`. The residual mention is a commented-out line in
  `init_object2.cpp`, so it clears with a tidy, not a migration.
- **xo-printjson** — the singleton's home; clears last.
- **xo-kalmanfilter** — `EigenUtil::provide_json_printers` from
  `init_filter.cpp`; the model the whole pattern was copied from, and the
  straightforward case.
- **xo-pyprintjson** — binds `PrintJsonSingleton::instance` as a python
  `.def_static`, so this one is an API change visible to python, not just a
  refactor. Decide what python callers get instead before migrating it.
- **xo-pyprocess, xo-pyreactor, xo-pywebsock** — python wrappers passing the
  singleton into `http_snapshot`/`http_endpoint_descr`; they need whatever
  xo-pyprintjson settles on.
- **xo-websock** — `utest/websock_utest_main.cpp` only, which is NOT BUILT as a
  test: its `add_test()` is commented out, "requires manual interaction from
  browser", and the file still uses v1-era quoted includes
  (`"printjson/PrintJson.hpp"`). Check whether it should be migrated or
  deleted before spending effort on it.

**Done when:**
- the symbol grep above returns empty
- `PrintJsonAppcx` constructs its own `PrintJson` rather than adopting the
  singleton, and `PrintJsonSingleton.{hpp,cpp}` are deleted
- `xo-build --sweep` green, and a json rendering test still passes in a binary
  that builds TWO contexts -- the property the singleton made impossible, and
  the one worth pinning once it is true
