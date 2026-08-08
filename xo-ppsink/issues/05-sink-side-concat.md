# 05 — a `tostr` replacement that concatenates *into* the sink

Status: fixed 2026-08-08
Type: task

`tostr(args...)` builds a `std::string`. When the only consumer is a sink, that
string is pure overhead — it is constructed solely to be handed straight back to
the sink that could have rendered the pieces directly.

The motivating shape, from the xo-expression/xo-reader pilot
(`.xo-backlog/xo-expression/issues/01-ppsink-migration-pilot.md`):

```cpp
for (std::size_t i = 0; i < z; ++i) {
    std::string i_str = tostr("[", z-i-1, "]");     // <- built to be thrown away
    st.field(i_str, stack_[i].get());
}
```

Wanted instead — something that carries the pieces and renders them where the
name is emitted, with no intermediate buffer:

```cpp
for (std::size_t i = 0; i < z; ++i)
    st.field(cat("[", z-i-1, "]"), stack_[i].get());
```

(`cat` is a placeholder name — see open questions.)

## Why it is worth more than it looks

**`tostr` is not a cheap concat.** Measured by reading
`xo-ppsink/include/xo/ppsink/tostr.hpp` (2026-08-08):

```cpp
template <typename... Ts>
std::string tostr(const Ts &... args) {
    std::stringstream ss;          // <- per call
    FlatSink sink(ss);
    (sink.pp(args), ...);
    return ss.str();               // <- plus a copy out
}
```

So each call constructs a `std::stringstream` (streambuf allocation, locale/ios
setup) and copies the result out. For a four-character label like `[12]` — which
would otherwise be free, being well inside SSO — the stringstream dominates
completely. In a loop over a container it is paid per element.

The header already flags the underlying concern and notes that an *arena-backed*
`tostr` cannot live in xo-ppsink, because ppsink sits below xo-arena in the level
order. **Sink-side concatenation sidesteps that constraint entirely**: there is
no buffer to allocate from, arena or otherwise, so it can live in ppsink.

## Scope — measured 2026-08-08

```bash
grep -rn "tostr(" xo-*/ --include=*.hpp --include=*.cpp | grep -v '\.build'
```

345 call sites tree-wide. Splitting them by what consumes the result (line-level
heuristic, so indicative rather than exact):

- **51** feed a tag/field/put on the same line — candidates for this facility
- **71** bind the result to a named `std::string` / `auto` / `return` — genuinely
  want a string, not candidates
- the remainder are error messages and one-off labels, mostly `throw
  std::runtime_error(tostr(...))`, which want a string

So this is **not** a mass migration: most `tostr` uses legitimately want a
string. The win concentrates in the loop cases, of which there are 5 today, all
in the pilot:

```
xo-expression/src/expression/Sequence.cpp:68,90,107
xo-reader/src/reader/exprstatestack.cpp:108
xo-reader/src/reader/envframestack.cpp:129
```

All five are the same `tostr("[", i, "]")` index-label idiom.

**Honest sizing:** 5 hot sites today. This is a correctness-of-design and
tidiness argument more than a measured performance problem — nobody has profiled
these paths, and they are diagnostic printing, not a hot loop. Worth doing
because the facet stack (233 files, many container-shaped) will multiply the
idiom, not because the current 5 are costing anything measurable.

## Open questions

- **Where does it attach?** `field()` currently takes `std::string_view name`
  and the builder emits it with `put()` (raw, unescaped, inside a
  `color_guard`). A composed name needs an overload that renders pieces in that
  same position, preserving rawness and emitting **no split** between pieces —
  the name must stay one unbreakable token.
- **Is it name-specific or general?** A general `cat(...)` usable anywhere a
  renderable is wanted (`xtag`, `put`, `field`) is more useful than a
  name-only overload, but has to interact with `Prettifier<>` dispatch rather
  than bypassing it.
- **Lifetime.** `cat()` would hold references to its arguments, same rule as
  `field()`. Safe within a full-expression, which covers the loop-body usage;
  worth the same prominent comment `field_impl` carries.
- **Naming.** `cat`, `concat`, `joined`, `lit`? Not `tostr_` — it is not a
  variant of tostr, it is the thing tostr should have been when the destination
  is a sink.
- **Does `pretty_struct`'s variadic form need it too?** It holds every field
  alive until the fold completes, so a temporary name there is also fine — but
  the same overload should work in both places for consistency.

**Files:**
- `xo-ppsink/include/xo/ppsink/tostr.hpp` — the current facility and its NB
- `xo-ppsink/include/xo/ppsink/pretty_struct.hpp` — `field()` / `field_impl`,
  where a name overload lands
- the 5 loop sites listed above — the first real consumers

**Done when:** — met
- [x] a sink-destined concat exists (`xo-ppsink/include/xo/ppsink/concat.hpp`)
      and all 5 loop sites use it
- [x] `tostr` left alone for the sites that genuinely want a `std::string`

## Resolution (2026-08-08)

`concat(args...)` returns a `concat_impl` holding **pointers** to its
arguments, with a `Prettifier` that folds `sink.pp` over them. Nothing is
buffered: the pieces render where the result is wanted.

Three differences from legacy `xo::concat`, all deliberate:

1. variadic, not binary
2. by reference, not by value — the whole point
3. **not atomic.** Legacy declared `concat_impl` a pretty-printing leaf
   ("don't want structure visible"). Here each argument goes through
   `sink.pp()`, so a structured argument would emit its structure. Fine for the
   short flat labels this exists for; documented in the header as a caveat.

### It needed two small API changes to be usable where it matters

`concat` alone was not enough — the motivating case is a *field name*, and
`field()` took `std::string_view`:

- `field_impl<Value>` → `field_impl<Name, Value>`, holding the name by pointer
  and rendering it with `sink.pp()` rather than `sink.put()`. Contained:
  `field_impl` is named nowhere outside `pretty_struct.hpp`, and all ~140
  `field("literal", v)` call sites keep working by deduction.
- `Prettifier<tag_impl>` likewise renders its name with `pp()`, so
  `xtag(concat(...), v)` works too — which is what let `Sequence::display()`
  drop its `std::string`.

For a string-like name `pp()` takes the same string-like leaf `put()` did, so
output is unchanged — confirmed by the existing `pretty_struct`/`struct_scope`
tests, which pin exact token streams and passed untouched.

### Converted sites

| | |
|---|---|
| `exprstatestack.cpp:99`, `envframestack.cpp:121` | `tostr(...)` → `concat(...)` as a `struct_open` field name |
| `Sequence.cpp:90` | same, in `pretty()` |
| `Sequence.cpp` `display()` | `xtag(concat("[", i, "]"), x)` — no intermediate string |

The remaining `tostr("[", ...)` hits tree-wide are in `xo-indentlog/utest`
(legacy, staying) and in concat's own test, which asserts the two agree.

### Tests

`xo-ppsink/utest/concat.test.cpp` — text, agreement with `tostr`, nesting,
string/string_view arguments, prvalue arguments, and two claims that needed
pinning specifically:

- **emits no structure of its own** (checked against the token stream via
  MarkSink; FlatSink cannot show this) — required if a concat is to stand in a
  field name, which must remain one unbreakable token
- **does not copy its arguments** (copy-counting type)

Both were mutation-checked: emitting a `split` between arguments, and switching
the tuple to hold values instead of pointers, each turn the suite red.

## Notes

Design it against the 5 real call sites rather than in the abstract — the
`hex_view` port showed how much the call sites constrain the API shape
(`.xo-backlog/xo-ppsink/issues/02-facility-gaps.md`).
