# 03 — InitEvidence is forgeable, so it documents intent rather than proving it

Status: open
Type: refactor

`InitEvidence` has a public converting constructor:

```bash
sed -n '/class InitEvidence/,/evidence()/p' xo-subsys/include/xo/subsys/Subsystem.hpp
```

so `InitEvidence{0}` is available to anyone.  A witness whose values can be
manufactured attests nothing: the type records an intention, and a reader who
takes it as proof is mistaken.

This matters more now that `02` has moved evidence into the appcx objects, where
it is meant to be load-bearing.  The appcx design leans on unforgeability twice:

- `FacetUtestAppcx` hands tests `main()`'s context specifically so a test cannot
  mint its own -- see `01`, where a forgeable config would let a test silently
  override an operator's command-line setting
- the python bindings render unconditionally (`Float.__repr__`,
  `AllocFlywheel.__repr__` via `TempPrettySink::pp2str`) on the argument that
  holding the object *is* proof the temp sink was configured

Both arguments are only as strong as the impossibility of fabricating the
evidence.

## Shape

- make `InitEvidence(std::uint64_t)` private, with `InitSubsys` (or a single
  named factory) its only friend
- keep the default ctor if a "no evidence yet" value is needed, or replace it
  with a named `InitEvidence::none()` so the empty case is deliberate
- `operator^=` stays: combining attestations is the useful operation and cannot
  create evidence that was not already held

**Blast radius:** the readers are test assertions, not production code:

```bash
grep -rn "\.evidence()" --include=*.cpp xo-*/utest | grep -v '/\.build/' | wc -l
```

**Done when:** a translation unit outside xo-subsys cannot construct an
`InitEvidence` carrying a non-zero value except by way of `InitSubsys`.
