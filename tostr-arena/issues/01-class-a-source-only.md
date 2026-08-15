# 01 — class A: source-only conversions (8 subsystems)

Status: open
Type: refactor
Progress: grep -rl '#include <xo/ppsink/tostr_xx\.hpp>' --include=*.hpp --include=*.cpp xo-alloc2/ xo-object2/ xo-gc/ xo-numeric/ xo-tokenizer2/ xo-expression2/ xo-reader2/ xo-pyjit/ 2>/dev/null | grep -v '/\.build/' | wc -l

Spec: `.xo-backlog/tostr-arena/spec.md`. Recipe and rationale live there; this
ticket is the scope and the counter.

**These eight already reach xo-indentlog2 and declare no `xo_ppsink` of their
own**, so the conversion is step 1 alone — include swap and call rename. No
CMakeLists change, no `pkgs/*.nix` change, no `subsystem-edges` re-capture.

In build order (`xo-build --list`):

| subsystem | files |
|---|---|
| xo-alloc2 | 1 |
| xo-object2 | 4 |
| xo-gc | 5 |
| xo-numeric | 1 |
| xo-tokenizer2 | 2 |
| xo-expression2 | 1 |
| xo-reader2 | 4 |
| xo-pyjit | 1 |

Re-derive rather than trusting the table (`CONVENTIONS.md` rule 2 — and the
counts move as work lands):

```bash
for s in xo-alloc2 xo-object2 xo-gc xo-numeric xo-tokenizer2 xo-expression2 xo-reader2 xo-pyjit; do
  xo-deps --why=$s:xo-indentlog2 -q >/dev/null 2>&1 || echo "$s NO LONGER class A: does not reach il2"
  grep -rl 'xo_ppsink' $s/CMakeLists.txt $s/src/*/CMakeLists.txt 2>/dev/null \
    && echo "$s NO LONGER class A: declares ppsink"
done
# (no output = the classification still holds)
```

## Do these two first

**xo-object2 (4) and xo-expression2 (1) are half-converted.** `638fd54a` and
`4eb66532` each moved a single file and left the rest, so both currently read as
converted in the commit log while most of their sites are not:

```bash
grep -rl 'ppsink/tostr_xx' --include=*.hpp --include=*.cpp xo-object2/ xo-expression2/ | grep -v '/\.build/'
grep -rl 'indentlog2/print/tostr' --include=*.hpp --include=*.cpp xo-object2/ xo-expression2/ | grep -v '/\.build/'
```

## The one file to look at rather than sed

`xo-alloc/include/xo/alloc/CircularBuffer.hpp` is **not** in this class (xo-alloc
is class B), so every site here is a `.cpp`. Verify that before bulk-editing —
if a public header turns up in this list, the subsystem has moved to class C's
problem and the conversion pushes indentlog2 onto its consumers:

```bash
grep -rl '#include <xo/ppsink/tostr_xx\.hpp>' --include=*.hpp \
  xo-alloc2/ xo-object2/ xo-gc/ xo-numeric/ xo-tokenizer2/ xo-expression2/ xo-reader2/ xo-pyjit/ \
  | grep -v '/\.build/' | grep '/include/'
# (expected: empty)
```

## Verification

Per subsystem:

```bash
xo-build -q --configure --with-utests --with-examples --build --install <sub>
nix-build ci.nix -A <sub> --no-out-link
```

`nix-build` earns its keep even though no nix file changes: it is the only check
that the *installed* package config still resolves, and these subsystems reach
indentlog2 transitively rather than by declaration, which is exactly the shape
that failed for xo-numeric during the earlier indentlog migration.

At the end of the class:

```bash
xo-build --sweep
#   62 attempted: 34 ok, 28 with no tests, 0 failed, 0 skipped
```

**No `subsystem-edges` re-capture** — nothing here changes a declaration. If a
re-capture *does* show a diff after this ticket, something in the class was
misclassified; go back to the loop above.

## Done when

- `Progress:` returns 0
- the sweep line is unchanged
- one commit per subsystem, matching the existing message shape
  (`xo-<sub>: upgrade to xo-indentlog2 tostr [REFACTOR]`)
