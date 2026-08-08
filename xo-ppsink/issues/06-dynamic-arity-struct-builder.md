# 06 — a struct builder for a runtime number of fields

Status: open (designed; not implemented)
Type: feature

`pretty_struct(name, fields...)` is variadic, so its arity is fixed at compile
time. There is currently **no way to render `<Name :f0 … :f1 … >` where the
number of fields is a runtime value** — i.e. a loop over a container.

Landing this before the xo-expression/xo-reader migration
(`.xo-backlog/xo-expression/issues/01-ppsink-migration-pilot.md`), so the API
arrives with its own tests rather than inside a 52-file refactor.

## Measured 2026-08-08

Neither existing facility covers the case:

- `pretty_struct.hpp` — `PpSink::pretty_struct(std::string_view name, const Fields &... fields)`.
  Compile-time pack; a loop cannot contribute to it.
- `PrettyVector.hpp` — `Prettifier<std::vector<T>>` renders `[a,b,c]`: no
  `<Name` prefix, no per-element field names.

## The design targets — three real call sites, quoted

These are the consumers the API must fit. They are *not* converted by this
ticket; they are the shape it is validated against. (`.xo-ppsink/issues/02`
records why a port wants a real consumer: the `hex_view` port was reshaped
substantially by its call sites.)

**1. `xo-expression/src/expression/Sequence.cpp` — loop, generated index labels**

```cpp
pps->write("<Sequence");
std::size_t i = 0;
for (const auto & expr_i : expr_v_) {
    std::string i_str = tostr("[", i, "]");
    pps->newline_pretty_tag(ppii.ci1(), i_str.c_str(), expr_i);
    ++i;
}
pps->write(">");
```

**2. `xo-reader/src/reader/exprstatestack.cpp` — fixed field + loop + forced break + reversed labels**

```cpp
if (ppii.upto()) {
    if (stack_.size() > 1)
        return false;              /* "always multiple lines if >1 element" */
    ...
} else {
    pps->write("<exprstatestack");
    pps->newline_pretty_tag(ppii.ci1(), "size", stack_.size());
    for (std::size_t i = 0, z = stack_.size(); i < z; ++i) {
        std::string i_str = tostr("[", z-i-1, "]");
        pps->newline_pretty_tag(ppii.ci1(), i_str, stack_[i].get());
    }
    pps->write(">");
}
```

**3. `xo-reader/src/reader/envframestack.cpp`** — structurally identical to (2),
same `if (stack_.size() > 1) return false;` policy.

Three requirements fall out: **runtime arity**; **a fixed field interleaved with
dynamic ones**; and **a forced break independent of margin**.

## Proposed shape

A scoped RAII builder that renders each field as it is added:

```cpp
{
    auto st = sink.struct_open("exprstatestack", /*force_break=*/ stack_.size() > 1);
    st.field("size", stack_.size());
    for (std::size_t i = 0, z = stack_.size(); i < z; ++i)
        st.field(tostr("[", z-i-1, "]"), stack_[i].get());
}   /* dtor: put(">") + end() */
```

- `struct_open(name, force_break=false)` emits `put("<") put(name) begin()`.
- `.field(name, value)` emits the separator then the field.
- destructor emits `put(">") end()`.
- `force_break` selects `newline()` instead of `split(1,0)` for the separator.
  `PpSink.hpp:153` documents `newline` as *"forced break: always newline +
  (running_indent + offset), **forcing every enclosing group to break**"* —
  exactly the legacy `return false`-before-printing policy.

**`pretty_struct` should become a thin variadic wrapper over the builder**, so
the four shape rules live in exactly one place. They are the rules that
`span_pp.hpp` and commit `e07b98b1` each got wrong:

1. `put(name)` **before** `begin()`, so the header stays on the opening line
2. `begin()` **no-arg** — indent comes from the sink's `PpConfig`
3. `split(1, 0)` **not** `split(1, indent)` — `begin()` already moved the running
   indent and `PpState` adds the split offset on top
4. a separator before **every** field, including the first

RAII additionally makes rule-4's counterpart — the closing `">"` inside the
group, before `end()` — impossible to forget.

## Tests: both layers, they catch different things

`pretty_struct` is already tested in both, and the split is load-bearing:

| | `xo-ppsink/utest` | `xo-indentlog2/utest` |
|---|---|---|
| sink | `FlatSink` + a recording sink rendering `<G>` / `<S s,o>` / `</G>` | `PrettySink` + `PpConfig().with_soft_right_margin(N)` |
| catches | shape-rule errors: name inside the group, missing separator, `">"` escaping the group | layout errors: indent double-counting, a group that will not break |

Both layers have caught a real bug in this migration — `e07b98b1` was a
token-stream error; `.xo-backlog/xo-ppsink/issues/03-nested-struct-layout.md`
was a rendering error the token stream looked fine for. Neither subsumes the
other.

**`xo-ppsink/utest/struct_builder.test.cpp`** (new)
- builder with a fixed field list emits the **same token stream** as
  `pretty_struct` with the same fields — proves the wrapper refactor is
  behaviour-preserving
- a loop-added field is indistinguishable from a variadic one
- zero fields → `"<P>"`
- `force_break=true` emits `newline` where `false` emits `split`
- a generated name (`tostr("[", i, "]")`) survives — no dangling view

**`xo-indentlog2/utest/struct_builder.test.cpp`** (new)
- `force_break=true` breaks at a margin **wide enough to fit** — the assertion
  that only exists here, since `newline()`'s "forces every enclosing group to
  break" contract is invisible in the token stream
- `force_break=false` still collapses to one line at the same margin
- a dynamically-built struct indents identically to a variadic one (2, not 4)
- nested: a builder inside a builder accumulates indent

Measure rendered output before writing expectations — predicting layout has
bitten this migration twice. `PrettySink` needs a unique `ArenaConfig` name per
call.

## Not in scope

- **Converting the three call sites.** They land with the pilot.
- **Sink-side concatenation** for the `tostr("[", i, "]")` names — separate,
  `.xo-backlog/xo-ppsink/issues/05-sink-side-concat.md`. This ticket should use
  plain `tostr` and let 05 improve it later.

## Open questions

- **Naming.** `struct_open` / `.field()` / RAII close? Or an explicit
  `.close()` with the dtor as a safety net? RAII-only is tighter but makes the
  emission point implicit at a scope end.
- **Does the builder need `present`?** `field()` already carries the flag;
  the builder can simply forward it. Probably free.
- **Move/copy.** The builder holds a `PpSink &` and must not be copied; whether
  it should be movable (to return one from a helper) is unclear — start
  non-copyable, non-movable.

**Inferred, not measured:** that this shape also fits the facet stack's
container-shaped types (233 files). Plausible — the same `<Name :field…>` idiom
runs through both — but nobody has checked, and it should not be treated as
established until someone does.

**Files:**
- `xo-ppsink/include/xo/ppsink/pretty_struct.hpp` — builder lands here;
  `pretty_struct` becomes its wrapper
- `xo-ppsink/include/xo/ppsink/PpSink.hpp:112` — the `pretty_struct` declaration
- `xo-ppsink/utest/`, `xo-indentlog2/utest/` — the two test layers

**Done when:**
- the builder exists, `pretty_struct` is implemented in terms of it, and its
  existing tests still pass unchanged
- both new test files pass
- the three design targets above are expressible without hand-rolling
  `begin`/`split`/`end`
