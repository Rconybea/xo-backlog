# 02 — InitEvidence lives in the config, so constructing a description performs an action

Status: open
Type: refactor

`FooConfig` constructors run subsystem initialization:

```bash
grep -rn "init_evidence_{InitSubsys" xo-*/src/*/[A-Z]*Config.cpp
```

One remaining as of 2026-09-07: `FacetConfig`.  `Indentlog2Config` has already
been converted -- its ctor now only stores its arguments, and the
`InitEvidence` moved to `Indentlog2Appcx`
(`Indentlog2Appcx.cpp:11`, `Indentlog2Appcx.hpp:51`), which is exactly the shape
this ticket argues for.  It is the worked example, and it builds:

```bash
grep -rn "init_evidence_{InitSubsys" xo-*/src/*/[A-Z]*Config.cpp   # FacetConfig only
grep -n "init_evidence_" xo-indentlog2/src/indentlog2/Indentlog2Appcx.cpp
```

So

```cpp
FacetConfig cfg{1024, 1024};   // <- initializes the xo-facet subsystem
```

is not a description of what is wanted; it is the act of arranging it.  This
ticket argues the evidence belongs in the **appcx**, and configs should be inert
data.

## Why

**1. Config construction has global side effects, and one of them cost an
afternoon (2026-09-07).**  While tracing a rendering failure, a staged probe
narrowed to a single statement: constructing a `FacetConfig` and nothing else
broke every `PrettySink` created afterwards.  The mechanism was
`InitSubsys<S_facet_tag>::require()` running from the config's member-init
list, and a debugging `scope` inside it writing to a sink that could not accept
output.  A config that were plain data could not have caused it.  Every step of
the bisect read as "constructing a description broke the printer", which is
precisely the confusion this shape creates.

**2. It blocks the direction in `01`.**  A reflection-based parser that
materializes configs from schematika literals would, as things stand,
initialize subsystems as a side effect of *parsing a file*, in whatever order
the file lists them.  Configs must be inert for that to work at all.  This is
the decisive argument: `01`'s destination requires this change first.

**3. Value semantics dilute the evidence.**  Configs are copyable and storable.
A copied `FacetConfig` carries an `InitEvidence` attesting nothing about that
copy's provenance.  Evidence should be as scarce as the work it records: one
appcx per process, many configs.

**4. It attests a weaker fact than callers need.**  `InitEvidence` in
`FacetConfig` records that `InitSubsys<S_facet_tag>::require()` ran.  What a
renderer needs is that an `Indentlog2Appcx` was *constructed* -- that is the
ctor which runs `TempPrettySink::init()` (`Indentlog2Appcx.cpp`).  Measured
2026-09-07: `InitSubsys<S_indentlog2_tag>::require()` alone leaves rendering
working, and is not sufficient for the appcx-dependent paths.  Config-held
evidence therefore cannot be the thing a `carries_indentlog2`-style constraint
returns.

## Shape

- `FooConfig`: drop `init_evidence_`; plain copyable data, no behaviour, no
  `InitSubsys` call -- as `Indentlog2Config` already does
- `FooAppcx`: run `InitSubsys<S_foo_tag>::require()` in its ctor and hold the
  result.  It is already the single point per process that takes the contexts
  below it, so ordering is checked where it already was
- `FacetAppcx`: additionally retain the `Indentlog2Appcx &` it is constructed
  with, which it presently discards ("Not used directly; proof of work") -- so
  the chain becomes navigable rather than existing only during construction

**Property being relocated, not added.**  Today config construction is what
forces initialization, so misordering is nearly impossible by accident.  After
this, the appcx ctor is solely responsible.  The utest mains are the callers
that would notice if it were dropped, so they are the regression surface.

**Files:**
- Modify: `xo-facet/{include/xo/facet/cx/FacetConfig.hpp, src/facet/FacetConfig.cpp}`
  and `src/facet/FacetAppcx.cpp` -- the remaining pair
- Check: `xo-interpreter2` has a third config/appcx pair; same treatment
- Reference: `xo-indentlog2` is done, and shows the target shape

**Done when:**
- constructing any `FooConfig` has no observable effect beyond storing its
  arguments
- `InitSubsys<S_foo_tag>::require()` runs exactly once per appcx construction
- the utest suites still pass, and `utest.facet` still initializes in the order
  `facet_utest_main.cpp` establishes

## Related

`InitEvidence` is currently forgeable -- `InitEvidence(std::uint64_t)` is a
public converting ctor (`xo-subsys/include/xo/subsys/Subsystem.hpp`), so anyone
can write `InitEvidence{0}`.  As long as that is true the type documents intent
rather than proving anything.  Making the ctor private, with `InitSubsys` its
only friend, would make it a witness in the sense `01` and the appcx design rely
on.  Worth doing with this ticket, since both are about evidence meaning what it
says.

Note the existing readers are assertions in tests:

```bash
grep -rn "\.evidence()" --include=*.cpp xo-*/utest | grep -v '/\.build/'
```

so tightening construction is unlikely to disturb production code.
