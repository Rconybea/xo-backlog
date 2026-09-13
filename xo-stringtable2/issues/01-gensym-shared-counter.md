# 01 — StringTable::gensym's counter is process-wide, not per-table

Status: diagnosed (2026-09-13)
Type: bug
Raised: RC, 2026-09-13, while asking whether xo-stringtable2 introduces any
singletons in the course of designing `Stringtable2Appcx`

It does not introduce one deliberately. It introduces this one by accident:

```cpp
const DUniqueString *
StringTable::gensym(std::string_view prefix)
{
    static std::size_t s_counter = 0;        // xo-stringtable2/src/stringtable2/StringTable.cpp:82
```

A function-local static, so **every `StringTable` in the process draws from one
sequence**. Each table is otherwise self-contained: it owns its `DArena` and its
`DArenaHashMap`, and is constructed with an explicit capacity
(`StringTable.hpp:33`). There is no `instance()`, and nothing else in the
subsystem is process-wide — the two `instance()` calls it makes reach other
subsystems' singletons (`CollectorTypeRegistry`, and `FacetRegistry` via
`SetupStringtable2::register_facets`).

## What it does and does not break

**Uniqueness is safe.** `gensym` loops until `lookup()` misses, so a name it
returns is unique *within its own table* whatever the counter does.

**Reproducibility is not.** Which name you get depends on how many gensyms every
other table in the process has done. Concretely, there is a live caller:

```bash
grep -rn 'gensym' --include=*.cpp --include=*.hpp xo-*/ | grep -v '/\.build/'
#   xo-reader2/src/reader2/ParserStateMachine.cpp:401:  return stringtable_.gensym(str);
#   xo-reader2/src/reader2/DLambdaSsm.cpp:351:          p_psm->gensym("lambda")
```

and `StringTable stringtable_;` is a **member** of `ParserStateMachine`
(`xo-reader2/include/xo/reader2/ParserStateMachine.hpp:380`) — so each parser
owns a table, and two parsers in one process produce `lambda:1`, `lambda:2` from
a shared run rather than each starting at `lambda:1`.

That is the shape that makes a test's expected output depend on test *ordering*:
a case asserting a gensym'd name passes alone and fails after another case has
parsed a lambda. No such test exists yet, which is the other half of the problem
— see below.

## Second defect on the same four lines, latent

```cpp
int n = snprintf(buf, sizeof(buf), "%s:%lu", prefix.data(), s_counter);
```

`%s` with `std::string_view::data()`. A `string_view` is **not** guaranteed
null-terminated, so this reads past the end for any view that is not. The
`assert(prefix.size() + 20 < sizeof(buf))` above bounds the prefix's length; it
does nothing about termination.

Latent today, not live: the only call site passes the literal `"lambda"`
(`DLambdaSsm.cpp:351`), which is terminated. It goes live the first time someone
passes a substring view — which the parameter type invites. `%.*s` with
`(int)prefix.size()` is the fix, and it removes the need for the assert to be
load-bearing.

## Why it went unnoticed

**`gensym` has no test at all**, in this subsystem or any other:

```bash
grep -rn 'gensym' xo-*/utest/ | grep -v '/\.build/'     # empty, 2026-09-13
```

`xo-stringtable2/utest/` has four files and covers `StringTable`'s `lookup`,
`intern` and `verify_ok` — `gensym` is the one method of the four that nothing
exercises. A per-table counter and a shared one behave identically in any test
that only ever makes one table, which is every test here.

## Files

- `xo-stringtable2/src/stringtable2/StringTable.cpp:78-106` — the whole method
- `xo-stringtable2/include/xo/stringtable2/StringTable.hpp:46` — declaration;
  the counter belongs beside `strings_`/`map_` at `:60-62`
- `xo-reader2/include/xo/reader2/ParserStateMachine.hpp:380` — the live caller's
  table, a member and therefore per-parser

## Done when

- [ ] the counter is a member of `StringTable`, not a function-local static
- [ ] `snprintf` uses `%.*s` with the view's size rather than `%s` with `.data()`
- [ ] a test makes TWO tables and asserts each gensym sequence starts from the
      same place — the case that fails today and cannot fail with one table
- [ ] a test passes a NON-null-terminated `string_view` prefix (e.g. a
      `substr` of a longer buffer) and asserts the result

## Notes

Worth keeping the reason this surfaced: the question was "does xo-stringtable2
introduce any singletons?", asked to decide what a `Stringtable2Appcx` would be
*for*, given every existing Appcx exists to tame one. The answer was "no" — and
then the grep for `static` found this. A subsystem can be free of singletons by
design and still have one by accident, and the accidental kind is the sort that
no Appcx will ever tame, because nothing declares it.
