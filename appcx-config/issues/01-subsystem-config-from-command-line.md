# 01 — subsystem configuration is hardcoded, not settable by the operator

Status: open
Type: feature

Every `FooConfig` value a process runs on is compiled in.  `facet_utest_main.cpp`
is representative:

```bash
grep -n "c_facet_registry_capacity\|c_type_registry_capacity\|c_temp_arena_capacity" \
    xo-facet/utest/facet_utest_main.cpp
```

three `constexpr` capacities, passed straight into `Indentlog2Config` and
`FacetConfig`.  So an operator cannot say "run this suite with a 64-entry facet
registry" to reproduce a capacity-exhaustion failure; the only way to change it
is to edit and rebuild.

The command line is already parsed -- `UtestAppStart::init(argc, argv)` runs
CLI11 and hands the remainder to catch2:

```bash
grep -n "add_flag\|add_option" xo-testutil/src/testutil/UtestAppStart.cpp
```

but it offers only `--debug`, `--announce`, `--help`.  Nothing reaches the
configs.

## Why the appcx design makes this worth doing

A setting is only worth having if it is **authoritative**, and that property
comes from the rule established in `FacetUtestAppcx`: tests receive the appcx
that `main()` built, rather than constructing one.

Had tests been able to mint their own `FacetConfig`, a `--facet-registry-capacity`
flag would be worse than absent.  The operator would pass it, `main()` would
honour it, the test would quietly build its own 1024, and the run would pass --
the knob appearing to work while doing nothing.  Because the appcx is threaded
instead, whatever `main()` puts in the config is what the process demonstrably
uses.  See `.xo-backlog/pyobject2/spec.md` for the same reasoning on the python
side, where a script is the host and `configure()` is its command line.

## Immediate shape

- `UtestAppStart` grows options for the capacities it can name, or a way to hand
  parsed values back to `main()`
- `facet_utest_main.cpp` builds `Indentlog2Config` / `FacetConfig` from those
  rather than from `constexpr` constants
- defaults come from `Indentlog2Config::make_default()` /
  `FacetConfig::make_default()`, so an unset flag and a c++ `main()` agree

**Done when:** `utest.facet --facet-registry-capacity=N` changes the capacity
the registry is actually built with, demonstrated by a test that observes it.

## Where this goes -- do not build the wide version first

The knob count is small today and will not stay that way:

```bash
# config variables currently declared
grep -hoE "uint32_t [a-z_]+_;" xo-*/include/xo/*/cx/*Config.hpp | wc -l
# subsystems with a config/appcx pair (three as of 2026-09-07)
find . -name "*Config.hpp" -path "*/cx/*" -not -path "*/.build/*"
```

Hand-written CLI11 options, one per variable per main(), is O(variables x
executables) and rots: each new subsystem config silently fails to be settable
until someone remembers to wire it.

The destination is a **parsed configuration representation** that every
subsystem's config is populated from -- reflection over the config types driving
the parse, with values supplied as schematika literals (or another notation that
reaches the same place), from a file and/or the command line.  Then adding a
config variable makes it settable by construction, and the same text configures
a utest binary, an application, and a python host.

That is a much larger piece of work and should not block the immediate one.  The
immediate work is compatible with it as long as `main()` keeps the shape
"obtain values, build FooConfig, construct the appcx, thread it" -- only the
first step changes when the parser arrives.
