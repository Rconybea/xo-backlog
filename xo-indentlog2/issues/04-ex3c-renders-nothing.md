# 04 — `ex3c` prints empty sections: reads `pp.output()` after PrettySink drained it

Status: diagnosed 2026-08-09 (deferred — RC 2026-08-09)
Type: bug

Small, and pre-existing. Found while checking that the `PpStyle` change had not
broken the colour examples — it had not; this was already broken.

Deferred deliberately: nothing depends on this example, the diagnosis below is
confirmed rather than a hypothesis, and the fix is three lines whenever someone
is next in the file. Filed so it is not rediscovered from scratch — the next
person to run `ex3c` will otherwise repeat the stash-and-rebuild comparison, and
that comparison has a trap in it (see below).

## Symptom

```bash
xo-indentlog2/.build/example/ex3c/indentlog2_ex3c
#   --- function_style = streamlined (via PrettySink) ---
#
#   --- function_style = simple (via PrettySink) ---
#   ...
```

Four headers, four empty bodies. The example exists to show function-name
styling and colour rendering through a real PrettySink, and shows neither.

Confirmed pre-existing, not caused by the `PpStyle` work: same output with the
change stashed and xo-ppsink + xo-indentlog2 rebuilt and reinstalled at HEAD.

**NB when checking this:** a stashed rebuild of only *some* subsystems leaves
stale objects against the other headers. `PpSink` changed size in that change,
so the first attempt at this comparison crashed with `std::length_error` — an
ABI mismatch, not a finding. `rm -rf <sub>/.build` before believing a
stash-and-rebuild result (`CONVENTIONS.md`, "Stale build artifacts lie").

## Diagnosis

`xo-indentlog2/example/ex3c/ex3c.cpp:65,71`:

```cpp
PrettySink pp(PpConfig().with_logbuf_config(logbuf_cfg), nullptr /*out*/);
...
return std::string(pp.output());
```

PrettySink drains each completed record to its attached sink and reclaims the
buffer, so `output()` only ever holds the *current, unfinished* record — empty
once the scopes have closed. With `nullptr` as the destination the drained
records go nowhere.

This is already written down, in `xo-indentlog2/utest/scope.test.cpp:29`:

> Capture via a dest streambuf (not pp.output()): PrettySink drains each
> completed record to the attached sink and reclaims its buffer, so output()
> only ever holds the current (unfinished) record.

So the test knows and the example does not.

## Fix

Give the PrettySink a real destination and capture that, as the test does:

```cpp
std::ostringstream oss;
PrettySink pp(PpConfig().with_logbuf_config(logbuf_cfg), oss.rdbuf());
...
return oss.str();
```

Check the sibling examples for the same shape while there — `ex3b`, `ex3f`,
`ex3g` (`ex3b` is fine: it renders and was verified to emit colour on
2026-08-09).

## Why it went unnoticed

An example that prints an empty string exits 0. Nothing in the build or the
test sweep looks at example *output*; `--with-examples` only proves they
compile. Same shape as the recurring failure this backlog keeps meeting:
absence looking like success.

**Files:**
- `xo-indentlog2/example/ex3c/ex3c.cpp:65,71`
- `xo-indentlog2/utest/scope.test.cpp:29` — the working pattern, and the
  comment that explains why

**Done when:**
- `ex3c` prints four styled, coloured renderings
- the other `ex3*` examples are checked for the same `pp.output()` misuse
