# 02 — `PrettyContext`: pretty-printing configuration as an object

Status: open
Type: feature

`.xo-backlog/xo-indentlog2/issues/01` added `toppstr(cfg, args...)`, which
takes a `PpConfig` and builds a `PrettySink` internally. That covers the
one-shot case, but leaves configuration as something threaded through call
sites rather than held.

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

- **Arena names must stay unique per sink.** `toppstr` currently mints one per
  call (`detail::toppstr_logbuf_config`), because two PrettySinks sharing an
  `ArenaConfig` name interfere and the symptom is wrong indentation in whichever
  renders second — a silent wrong answer. If a single `PrettyContext` can make
  many sinks, `make_sink()` inherits that duty; it must not simply hand out the
  stored `PpConfig`. Pinned today by
  `xo-indentlog2/utest/toppstr.test.cpp`'s `toppstr-is-repeatable`, which should
  be extended to cover repeated `make_sink()` from one context.
- **`ArenaConfig::size_` defaults to 0 and a zero-sized logbuf aborts** — see
  issue 01. The default ctor must supply the 64k default, not merely
  `PpConfig()`.
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
