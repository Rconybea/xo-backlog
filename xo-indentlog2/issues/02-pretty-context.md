# 02 — `PrettyContext`: pretty-printing configuration as an object

Status: open
Type: feature

`.xo-backlog/xo-indentlog2/issues/01` added `toppstr(cfg, args...)`, which
takes a `PpConfig` and builds a `PrettySink` internally. That covers the
one-shot case, but leaves configuration as something threaded through call
sites rather than held.

**Partly overtaken by `522799d7` (RC, 2026-08-09), "xo-indentlog2: streamline
scratch PpSink/PpConfig".** That commit added named `PpConfig` factories —
`plain()`, `colored()`, `scratch(margin)`, `scratch_aux(basename, margin,
style)` — and made `toppstr(cfg, ...)` use the caller's config **as given**
rather than rewriting its logbuf name. Two consequences for this ticket, both
detailed under *Care needed*:

- the "configuration as a value you can build in one expression" motivation is
  now largely delivered by the factories, so what is left that only a
  `PrettyContext` gives is the `std::streambuf *` and the `ThreadPrettySink`
  relationship;
- the arena-name invariant **moved**, and moved in a direction that makes
  `make_sink()` harder rather than easier.

**Proposed by RC 2026-08-09.** Introduce a `PrettyContext` holding the two
things a PrettySink is built from:

```cpp
class PrettyContext {
public:
    PrettyContext();                                   // default: 64k logbuf
    PrettyContext(const PpConfig & cfg, std::streambuf * out);

    PrettySink make_sink() const;

    template <typename... Ts>
    std::string toppstr(const Ts &... args) const;
};
```

and the free function becomes a thin wrapper:

```cpp
template <typename... Ts>
std::string toppstr(const Ts &... args) {
    PrettyContext cx;
    return cx.toppstr(args...);
}
```

## Why

**Configuration becomes a value you can hold, pass and vary** — margin today,
and whatever pretty-printing behaviour is added later (break policy, colour,
function-name styling) without changing every call site's signature. Right now
each knob has to reach the sink through a `PpConfig` argument threaded down, or
be defaulted and inaccessible.

The `std::streambuf *` argument is the part `toppstr` currently cannot express
at all: it hardcodes `nullptr` and returns `.output()`. Measured 2026-08-09:

```bash
grep -rn 'PrettySink' -A3 xo-interpreter/src/interpreter/Schematika.cpp \
    xo-interpreter2/src/skrepl/skreplxx.cpp \
    xo-reader/examples/exprreplxx/exprreplxx.cpp
```

- `Schematika.cpp:36` — `PrettySink pp(cfg, nullptr)`, takes `.output()`
- `exprreplxx.cpp:76` — likewise
- `skreplxx.cpp:174` — passes a real `clog.rdbuf()`, with a 1MB logbuf

So both shapes are live in production, and a context that carries the
streambuf covers both where `toppstr` covers only one.

## Relationship to ThreadPrettySink

`ThreadPrettySink::thread_install_once(cfg, out)`
(`xo-indentlog2/include/xo/indentlog2/print/PrettySink.hpp:91`) already
installs a thread-wide sink from exactly the same two ingredients. **Settle how
these relate before implementing** — a `PrettyContext` that duplicates it
without subsuming or composing with it would leave two ways to say the same
thing.

Plausible readings, not yet decided:

- `PrettyContext` is the value; `ThreadPrettySink` becomes "install *this*
  context for the thread"
- `ThreadPrettySink` stays the ambient case and `PrettyContext` is strictly the
  explicit, scoped one

## Care needed

- **Arena names must stay unique per sink** — because two PrettySinks sharing an
  `ArenaConfig` name interfere and the symptom is wrong indentation in whichever
  renders second, a silent wrong answer rather than an error.

  **Where that duty lives changed in `522799d7`.** It used to sit in
  `toppstr()`, which minted a fresh name per call
  (`detail::toppstr_logbuf_config`, now deleted) and so *could not* be given a
  colliding config. It now sits in `PpConfig::scratch_aux()`, which mints the
  name **once, when the config is built** — and `toppstr()` passes the config
  straight through. So a `PpConfig` is now a value carrying a specific arena
  name, and re-using one is the caller's business.

  That makes `make_sink()` *more* dangerous, not less: a `PrettyContext` holding
  one `scratch()` config and handing it to many sinks gives them all the same
  name. `make_sink()` must re-mint (`PpConfig::with_logbuf_name`, added by the
  same commit, is the tool), and this ticket's test obligation stands.

  Note the current tests do not falsify the hazard: `toppstr-carries-style`
  (rewritten by `522799d7`) renders twice from **one** `PpConfig::plain()` and
  passes. Sequential re-use of a name is evidently fine; what is unproven is
  concurrent or nested use. Worth establishing before `make_sink()` is designed
  around it — the whole bullet may be smaller than it looks.
- ~~**`ArenaConfig::size_` defaults to 0 and a zero-sized logbuf aborts.**~~
  Answered by `522799d7`: `PpConfig::scratch_aux()` sets 64k, so
  `PpConfig::plain()` / `scratch()` are safe to hand to a `PrettySink` where a
  bare `PpConfig()` still is not. A `PrettyContext` default ctor should delegate
  to `PpConfig::plain()`.
- **`make_sink()` returning by value** needs `PrettySink` to be movable, and it
  owns arena state; check before committing to that signature.

**Files:**
- `xo-indentlog2/include/xo/indentlog2/print/toppstr.hpp` — the free function
  that becomes a wrapper
- `xo-indentlog2/include/xo/indentlog2/print/PrettySink.hpp` — `PrettySink` and
  `ThreadPrettySink`
- the three production call sites above, as the first consumers worth migrating

**Done when:**
- `PrettyContext` exists, with the default ctor giving today's `toppstr`
  behaviour
- free `toppstr` delegates to it, and its tests still pass unchanged
- a decision is recorded on the `ThreadPrettySink` relationship
- repeated `make_sink()` from one context cannot collide on arena names, pinned
  by test

## Notes

Do not let this become a reason to defer phase C of
`.xo-backlog/xo-printable2/issues/01`. That work needs only `toppstr(cfg, x)`,
which exists; `PrettyContext` is a generalisation that can land before, during
or after, and its main consumer is the REPL rendering follow-on in
`.xo-backlog/xo-interpreter/issues/02`.
