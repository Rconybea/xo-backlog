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

## Two defects, not one — corrected 2026-08-22

### A. What you actually see today: the default log sink is FlatSink

`scope` logging goes to a process-wide `FlatSink` unless a `PrettySink` is
explicitly installed (`xo-ppsink/src/ppsink/LogState.cpp:11-16`, comment:
*"POC: whole-program FlatSink"*). And `FlatSink` drops the indent by design
(`xo-ppsink/src/ppsink/FlatSink.cpp:94`):

```cpp
FlatSink::newline(int32_t /*offset*/)
{
    /* a forced break is a hard newline even in flat output (no indent) */
    sbuf_->sputc('\n');
    return *this;
}
```

That is the column-0 output above, in full. Note the asymmetry with its
neighbour `FlatSink::split` (`:85`), which *does* stay flat -- it renders a
split as its flat spaces and never breaks. So a flat sink refuses soft breaks
but honours forced ones, and honours them without indent. Either half is
defensible; together they produce ragged output.

`PrettySink` is installed in only four places tree-wide -- `xo-alloc2`'s test
main, `scope.test.cpp`, and the ex3 examples:

```bash
grep -rn 'thread_install_once\|log_set_sink' --include=*.cpp xo-*/ | grep -v '/\.build/'
```

So essentially all scope logging in xo hits path A.

### B. With a PrettySink installed: relative indent works, the base does not

Same nested structs, `PrettySink` installed, drained to a streambuf:

```
                  <Outer          <- scope prefix puts this at column 18
    :o1 1                         <- continuation at 4, not 18
    <Inner
      :i1 1                       <- nested one deeper: 6.  relative indent OK
      :i2 2>
    :o2 2>
```

Group-relative indentation is correct. The **base** is not: continuation lines
start at the group indent measured from column 0, ignoring the log-line prefix.
Confirmed by running the same log at scope depth 1, where the prefix is wider
and the continuation lines do not move.

The prefix is written *through the sink* (`xo-ppsink/src/ppsink/scope.cpp:14`,
`st.sink().put(pad)`, and `emit_time(sink, ..)`), so the column is known to the
sink -- `PrettySink` even implements `lpos()` over `logbuf_.viz_lpos()`
(`PrettySink.hpp:83`). `PpState::print_indent_` simply starts each record at 0
and never learns of it.

**Suggested fix for B, unverified:** seed `print_indent_` from `lpos()` at
record start. That also makes the `fits` calculation honest, since it would then
know the true starting column -- today a group is measured as if it began at
column 0, so it over-estimates how much fits.

## The first diagnosis was wrong — kept per rule 6

This ticket originally blamed `PpState::newline()` / `LogBufferAdapter::
newline_indent`, citing `PpState.cpp:232`, `:461`, `:464` and
`LogBuffer.cpp:111`. **None of that code ran in the observed case.** The demos
used the default sink, which is `FlatSink`; `PpState` and `LogBuffer` belong to
`PrettySink`.

Why it was plausible: the reasoning about `PpState` was itself correct --
`newline(offset)` really does defer to `split(0, offset)` and really does render
at `print_indent_ + tk_offset`, so the offset is already relative and RC's
proposed "make newline relative" change really would have been a no-op. Every
step checked out. The unchecked step was *which sink was executing*, and nothing
in ragged output points at a sink.

The falsifying command is one line, and the ticket already contained the hint:
it recorded that driving a bare `PrettySink` tripped
`assert(false)` in `write_span`, i.e. that the sink under suspicion was not even
usable the way the demo used it. That was evidence about sink identity, read as
an inconvenience.

```bash
grep -n -A6 'require_default_sink' xo-ppsink/src/ppsink/LogState.cpp
```

Same shape as the three failures `CONVENTIONS.md` opens with: a claim about
*why*, built on evidence about *what*.

## Why it went unnoticed

It needs a struct that **forces** a break, inside a log record. Soft breaks
cannot expose it: on `FlatSink` a split stays flat by design, and on
`PrettySink` a split only breaks when the group overruns the margin, which
short diagnostic structs rarely do. `force_break` is the one construct that
breaks unconditionally, and it is newer than both sinks.

Careful: "FlatSink never breaks" is **false** and is worth not repeating -- it
is what makes defect A surprising. `FlatSink::split` stays flat;
`FlatSink::newline` emits a real newline. Only the split half is flat.

Expect it to become MORE visible as `ostream-containment` proceeds: that
milestone's whole purpose is moving printers off flat inserters and onto
`PpSink`, so the population of structs that can force a break inside a scope
only grows. Every `pretty()` written during that sweep is a candidate.

## Blast radius

Every force-broken struct logged inside a scope, tree-wide -- i.e. most
structured logging in xo, since defect A applies to the default sink and
`PrettySink` is installed in only four places. Anything using
`struct_open(name, true)`, or `pretty_struct` on a group that overruns the
margin.

## Files

**A (default sink):**
- `xo-ppsink/src/ppsink/FlatSink.cpp:94` — `newline()`: `'\n'`, no indent
- `xo-ppsink/src/ppsink/FlatSink.cpp:85` — `split()`: stays flat, for contrast
- `xo-ppsink/src/ppsink/LogState.cpp:11-16` — `FlatSink` as process-wide default

**B (PrettySink base indent):**
- `xo-indentlog2/src/indentlog2/PpState.cpp:461` — `off = print_indent_ + tk_offset()`
- `xo-indentlog2/src/indentlog2/PpState.cpp:464` — `p_out_->newline_indent(indent_z)`
- `xo-ppsink/src/ppsink/scope.cpp:14` — the prefix, written through the sink
- `xo-indentlog2/include/xo/indentlog2/print/PrettySink.hpp:83` — `lpos()`, the column source

**Shared:**
- `xo-ppsink/include/xo/ppsink/pretty_struct.hpp:228` — `separator()` -> `newline(0)`
- `xo-indentlog2/utest/PrettySink.test.cpp:105` — standalone indent expectation, passes

## Also observed

Reproducing B required draining the sink to a streambuf: constructing
`PrettySink(cfg, nullptr)` and reading `pp.output()` -- the pattern `ex3c` uses
-- returned an empty string. That is `xo-indentlog2/issues/04-ex3c-renders-nothing.md`,
independently reproduced here.

## Done when

- **A decided**: either `FlatSink::newline` indents forced breaks, or it stops
  honouring them (staying flat, as its `split` already does), or scope logging
  installs a `PrettySink` by default. Not left as-is by inattention.
- **B fixed**: with a `PrettySink` installed, a force-broken struct logged
  inside a scope indents its continuation lines to at least the scope's own
  column, and moves when scope depth changes.
- a test pins it at **non-zero** starting column -- the existing table in
  `PrettySink.test.cpp` starts at column 0, which is exactly the case that
  already passes.
