# 03 — InitEvidence is forgeable, so it documents intent rather than proving it

Status: fixed 2026-09-07
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

- make `InitEvidence(std::uint64_t)` private, with **`SubsystemImpl<Tag>`** its
  only friend.

  NB not `InitSubsys<Tag>`, as this ticket first said.  No `InitSubsys`
  specialization constructs evidence -- they only XOR what
  `Subsystem::provide<Tag>()` returns, and the two producers are both in
  Subsystem.hpp.  It matters because each subsystem writes its own
  `InitSubsys<Tag>` specialization in its own header (that is the extension
  point), so befriending that template would have granted forging rights to
  anyone who declares a tag.  Befriending `SubsystemImpl` keeps the producer
  single and closed.
- keep the default ctor if a "no evidence yet" value is needed, or replace it
  with a named `InitEvidence::none()` so the empty case is deliberate
- `operator^=` stays: combining attestations is the useful operation and cannot
  create evidence that was not already held

**Blast radius:** the readers are test assertions, not production code:

```bash
grep -rn "\.evidence()" --include=*.cpp xo-*/utest | grep -v '/\.build/' | wc -l
```

**Done when:** met 2026-09-07.  Verified by compile probe -- an outside TU
writing `xo::InitEvidence(42)` is rejected:

```
error: 'xo::InitEvidence::InitEvidence(uint64_t)' is private within this context
```

while `InitEvidence()` and `a ^ b` still compile, and
utest.{indentlog2,facet,object2,interpreter2,expression2} all pass -- the last
of which is where most of the 61 `.evidence()` assertions live.
