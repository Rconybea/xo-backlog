# 06 — `s_ppconfig_seq`: the scratch-arena counter lost `static` and lost atomicity

Status: diagnosed 2026-08-09
Type: bug (latent)

`522799d7` consolidated two arena-name counters into one:

```cpp
// xo-indentlog2/src/indentlog2/PpConfig.cpp:101
int s_ppconfig_seq = 0;
```

It replaced:

- `static std::atomic<int> s_seq{0};` — function-local, in
  `detail::toppstr_logbuf_config` (`toppstr.hpp`, deleted by the same commit)
- `static int s_seq = 0;` — function-local, in `PrettySink::scratch`
  (`PrettySink.cpp:40`)

Two properties were lost, one from each predecessor.

## 1. External linkage

`s_ppconfig_seq` is at namespace scope in `xo::pp` with no `static` and no
anonymous namespace, so `xo::pp::s_ppconfig_seq` is an exported symbol of
`libxo_indentlog2.so`. Both predecessors were function-local. Nothing depends on
it being visible; the `s_` prefix says it was not meant to be.

```bash
nm -D --defined-only ~/local/lib/libxo_indentlog2.so | grep ppconfig_seq
```

## 2. Atomicity

The `toppstr` counter was `std::atomic<int>`, deliberately: the comment it
carried explained that two `PrettySink`s sharing an `ArenaConfig` name interfere
and **the symptom is wrong indentation in whichever renders second — a silent
wrong answer, not an error** (recorded in issues 01 and 02). `++s_ppconfig_seq`
on a plain `int` is a data race, and two threads can now be handed the same
scratch arena name.

Partial regression, not a new class of bug: `PrettySink::scratch`'s counter was
already non-atomic, so the hazard existed on that path. Consolidating onto the
weaker of the two is the wrong direction.

## Fix

```cpp
namespace { std::atomic<int> s_ppconfig_seq{0}; }
```

and `s_ppconfig_seq.fetch_add(1) + 1` at the use site. One line each; no
interface change.

## Worth checking while there

`PpConfig::scratch_aux`'s default basename is `"anon"`, so every
`PpConfig::plain()` / `scratch()` arena is `anon1`, `anon2`, … in one global
sequence. That is fine for uniqueness but loses the provenance the old
`"xo.toppstr.N"` name carried, which is what a leak or overflow report would
show. Cheap to keep: make the default `"xo.pp.anon"`.

**Files:**
- `xo-indentlog2/src/indentlog2/PpConfig.cpp:101,108` — the counter and its use

**Done when:** the counter is internal-linkage and atomic, and the tree still
sweeps clean.
