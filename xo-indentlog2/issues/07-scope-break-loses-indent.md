# 07 — a force-broken struct logged inside a scope loses its indent

Status: diagnosed 2026-08-22
Type: bug
Milestone: ostream-containment

A struct rendered with `force_break` puts its opening line at the scope's
indent and every continuation line at **column 0**, with no log-line prefix:

```
11:14:50.820492   +(1) inner inner
                    <TypeRegistry
:[0] Alpha
:[2] Beta
:[3] Gamma>
```

Reproduced at two different scope depths, and with a plain three-field struct
that has nothing to do with the registries above, so it is not specific to any
one Prettifier:

```cpp
struct Demo {};
namespace xo::pp {
    template <> struct Prettifier<Demo> {
        static void print(PpSink & sink, const Demo &) {
            auto st = sink.struct_open("Demo", true /*force_break*/);
            st.field("alpha", 1);
            st.field("beta", 2);
        }
    };
}
int main() {
    scope log(XO_DEBUG_(true), "outer");
    log && log(Demo{});           // continuation lines land at column 0
}
```

`force_break = false` renders correctly (`<Demo :alpha 1 :beta 2>` at the scope
indent), because nothing breaks.

## Not what it looks like

The obvious suspect is `newline()`'s offset semantics -- that the argument
should be relative to the running indent rather than absolute. **It already
is.** `PpState::newline(offset)` marks every open group must-break and defers to
a split (`PpState.cpp:232`):

```cpp
this->split(0 /*spaces*/, offset);
```

and the break renders relative to the running indent (`PpState.cpp:461`):

```cpp
int32_t off = print_indent_ + token->tk_offset();
```

So `newline(0)` already means "keep the current indent" and `newline(-2)` would
already outdent. Changing that is not the fix.

**The indent machinery also works standalone**, pinned by this subsystem's own
test (`xo-indentlog2/utest/PrettySink.test.cpp:105`):

```cpp
{ begin(), stream("foo,"), split(), stream("bar"), end() }  ->  "foo,\n  bar"
```

`indent_width` 2 at nesting depth 1 gives newline + 2 spaces. That expectation
would fail if `print_indent_` were not accumulating or the offset were not
relative.

## What it is

The indent visible on a scope line is a **per-line prefix** -- timestamp plus
scope nesting -- written by the logging layer, not by `PpState`. When a break
occurs mid-record, `PpState` calls `p_out_->newline_indent(indent_z)`
(`PpState.cpp:464`) and `LogBufferAdapter::newline_indent`
(`LogBuffer.cpp:111`) emits `'\n'` followed by `indent_z` spaces. It does not
re-emit the prefix, and `print_indent_` has no idea the prefix column exists.

So the two indent systems are unaware of each other: `PpState` indents from
column 0, the log layer indents by writing a prefix, and a break inside a record
gets the former without the latter.

**Unverified**, and it is the thing to settle first: whether the fix is (a) the
log layer re-emitting its prefix (or a blank of equal width) on an internal
newline, or (b) seeding `print_indent_` with the prefix column at record start,
via `PpSink::lpos()` which `PrettySink` already implements
(`PrettySink.hpp:83`). (b) looks cheaper and keeps the wrapping decision honest
-- `fits` calculations would then also know the true column -- but it has not
been tried.

I could not isolate this with a bare `PrettySink`: driving one directly trips
`assert(false)` in `LogBufferAdapter::write_span` (`LogBuffer.cpp:154`), since
it expects the record/drain protocol. The diagnosis therefore rests on the two
scope observations plus the standalone test above, not on a probe.

## Why it went unnoticed

Three conditions have to coincide: a struct that breaks, inside a log record,
rendered by a sink that can break at all. `FlatSink` never breaks, so every
`pp_to_stream()` / `*_ostream.hpp` path is immune -- which is most of what has
been converted so far. `force_break` is also newer than the sinks.

Expect it to become MORE visible as `ostream-containment` proceeds: the whole
point of that milestone is moving printers off flat inserters and onto
`PpSink`, so the population of structs that can break inside a scope only grows.

## Blast radius

Every force-broken struct logged inside a scope, tree-wide -- i.e. most
structured logging in xo. Anything using `struct_open(name, true)` or
`pretty_struct` where the group does not fit.

## Files

- `xo-indentlog2/src/indentlog2/PpState.cpp:232` — `newline()` -> `split(0, offset)`
- `xo-indentlog2/src/indentlog2/PpState.cpp:461` — `off = print_indent_ + tk_offset()`
- `xo-indentlog2/src/indentlog2/PpState.cpp:464` — `p_out_->newline_indent(indent_z)`
- `xo-indentlog2/src/indentlog2/LogBuffer.cpp:111` — `newline_indent`: `'\n'` + spaces, no prefix
- `xo-ppsink/include/xo/ppsink/pretty_struct.hpp:228` — `struct_scope::separator()` -> `sink_.newline(0)`
- `xo-indentlog2/utest/PrettySink.test.cpp:105` — the standalone indent expectation

## Done when

- a force-broken struct logged inside a scope indents its continuation lines to
  at least the scope's own column
- a test pins it at **non-zero** starting column -- the existing table starts at
  column 0, which is exactly the case that already passes
