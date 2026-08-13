# 01 — DVsmApplyClosureFrame prints its environment as a raw address

Status: fixed 2026-08-13
Type: bug
Progress: grep -rn 'refrtag("env", local_env_)\|field("env", local_env_)' xo-interpreter2/src --include=*.cpp | grep -v '/\.build/' | wc -l

`DVsmApplyClosureFrame`'s printer emits its `DLocalEnv *` child as a **pointer
address**, not as an environment:

```
<DVsmApplyClosureFrame :cont apply_cont :env 0x7879d025b098>
```

and, when the pointer is null, as `0` rather than `nullptr`.

Observed 2026-08-12 while converting the printer to ppsink
(`.xo-backlog/xo-printable2/issues/01-aprintable-pretty-ppsink.md`). Both the
deprecated and the ppsink rendering are shown above because **they are
identical** — see "Not a divergence" below.

## Why

`local_env_` is a bare `DLocalEnv *`, and nothing teaches either pretty-printer
what to do with that type:

```bash
grep -rn 'ppdetail<.*DLocalEnv\|Prettifier<.*DLocalEnv' \
  xo-interpreter2/ --include=*.hpp --include=*.cpp | grep -v '/\.build/'
```

returns nothing (rc=1). So both protocols fall through to their leaf fallback,
which is `operator<<` on a raw pointer — hence the address, and hence `0` for
null.

Contrast `DClosure`, in the same subsystem, which renders the **same type**
correctly (`xo-interpreter2/src/interpreter2/DClosure.cpp`):

```cpp
obj<APrintable,DLocalEnv> env_pr(const_cast<DLocalEnv *>(env_));
...
field("env", env_pr, bool(env_pr))
```

It wraps the pointer in `obj<APrintable,DLocalEnv>` first, so the field renders
`<DLocalEnv :n_args 1>`. `DVsmApplyClosureFrame` passes the raw pointer
straight to the tag. That is the whole difference.

## Not a divergence — which is why it was not fixed in the refactor

The ppsink conversion reproduced the field literally. This is deliberate:

- the two protocols agree **byte for byte**, address included, so there is no
  behaviour change to justify under a behaviour-preserving refactor
- unlike `DApplyExpr`'s missing separator (recorded in the ppsink ticket), where
  `struct_open()` happened to emit the correct form for free, here nothing in
  ppsink improves it by accident — "fixing" it would be an intentional
  behaviour change smuggled into a mechanical conversion

Pinned in `xo-interpreter2/utest/printable_render.test.cpp`
(`interpreter2-applyclosureframe-render`) with the address scrubbed to `0xADDR`
by a `scrub_addr()` helper, in the same spirit as `scrub_type_id` /
`scrub_tseq` elsewhere. **The need to scrub is itself the evidence** that the
field carries nothing a reader can use: it is not merely ugly, it is
unpinnable, and it changes every run.

## Fixing it

One line, once someone decides the rendering should change:

```cpp
obj<APrintable,DLocalEnv> env_pr(local_env_);
...
field("env", env_pr, bool(env_pr))
```

matching `DClosure` exactly. Then:

- the four expectations in `interpreter2-applyclosureframe-render` change, and
  `scrub_addr()` and its `<cctype>` include can be deleted if no other field
  needs them — check first with
  `grep -n scrub_addr xo-interpreter2/utest/printable_render.test.cpp`
- do it **after** phase E deletes `pretty_deprecated`, or the two protocols stop
  agreeing and the deprecated expectations have to carry the old address form
  until they are deleted anyway

Note the null case is load-bearing: `local_env_` **can** be null in practice
(the test constructs one that way and it renders), so whatever replaces the
field must keep a present flag or an explicit null rendering rather than
dereferencing.

## Fixed 2026-08-13

Done as prescribed, after phase E removed the deprecated protocol:

```cpp
obj<APrintable,DLocalEnv> env_pr(local_env_);
...
field("env", env_pr, bool(env_pr))
```

`:env` now renders `<DLocalEnv :n_args 1>` where it printed
`0x7879d025b098`. `scrub_addr()` and its `<cctype>` include are gone from
`xo-interpreter2/utest/printable_render.test.cpp` — nothing in that file
renders an address any more.

**One behaviour change beyond the obvious.** A null `local_env_` printed
`:env 0`; it now **omits the field entirely**, because the present flag drops
it. That is what `DClosure` does for the same type, and it is deliberate — but
it is a different rendering from "renders zero", not merely a prettier one, so
the `null-env` cases pin it explicitly.

Mutation-checked four ways, all caught: forcing the present flag true, forcing
it false, renaming the tag, and — the one that matters — **reverting the field
to the raw `local_env_`**, i.e. regressing to the address form.

Verified: interpreter2 suite 177 assertions in 22 test cases;
`xo-build --sweep` → `62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped`;
`nix-build ci.nix -A xo-interpreter2 --no-out-link` green.

### A process note on the edit itself

The first attempt to update the expectations was a **silent no-op**: the
replacement searched for the two-expectation form (`expect_deprecated` +
`expect_pretty`), which phase E had already collapsed to one. The script
reported success and the test then failed against the old `0xADDR` string.
Cheap to spot here because a test caught it — but the lesson is general and
recurred several times across phase E: **a scripted source edit must assert
that its pattern matched**, or it reports success for having done nothing.

## Wider question

Whether a raw `D*` pointer should be printable at all is the same question
`.xo-backlog/xo-reader2/issues/01-ssm-printer-null-children.md` raises from the
other side. There, a pointer reached a printer through an empty `obj<>` and
**aborted**; here, one reaches a printer as a raw pointer and **silently prints
garbage**. Both would be caught by making the fallback refuse pointer types it
has no `Prettifier<>` for, rather than falling through to `operator<<`.

That is a change to xo-ppsink, not to this subsystem, and it would surface
every other site of the same shape at once. Worth considering when
`.xo-backlog/xo-ppsink/issues/02-facility-gaps.md` is next opened.
