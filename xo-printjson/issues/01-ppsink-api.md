# 01 — PrintJson API: take a PpSink instead of a std::ostream

Status: open
Type: task

`xo-printjson` serialises to `std::ostream *` throughout. Change it to write
into a `xo::pp::PpSink`.

**Motivation (RC): if we are printing JSON, the option to prettify it may be
useful.** That is a stronger case than the usual migration argument. JSON has
real nested structure, and indented/wrapped JSON is a thing people actually
want — which is exactly what a PpSink provides and a raw ostream cannot. The
serialiser already knows where every object and array boundary is; it just
throws that structure away as it writes.

Deferred out of the xo-webutil PpSink refactor (2026-08-06), which was scoped to
diagnostic printing only.

## Current API surface

`xo-printjson/include/xo/printjson/PrintJson.hpp`:

```cpp
template <typename T>
void print(T const & x_arg, std::ostream * p_os) const;   // :31
void print_tp(TaggedPtr tp, std::ostream * p_os) const;   // :39
void print_obj(rp<SelfTagging> const & obj, std::ostream * p_os) const;  // :44
void print_aux(TaggedPtr tp, std::ostream * p_os) const;  // :55
```

`xo-printjson/include/xo/printjson/JsonPrinter.hpp`:

```cpp
virtual void print_json(TaggedPtr tp, std::ostream * p_os) const = 0;   // :32
void report_internal_type_consistency_error(TypeDescr, TypeDescr,
                                            std::ostream * p_os) const;
```

**`print_json` is a pure virtual with 9 subclasses tree-wide** — that is the
bulk of the change, and it cannot be done incrementally per-subclass without
carrying both signatures for a while.

## Blast radius

Subsystems naming PrintJson: **xo-websock (5 files), xo-kalmanfilter (3),
xo-reflect, xo-reactor, xo-process, xo-pywebsock, xo-pyreactor, xo-pyprocess,
xo-pyprintjson** — plus xo-printjson itself (7).

The awkward consumer is **`xo-reactor/include/xo/reactor/EventStore.hpp:67`**,
where an `HttpEndpointDescr`'s endpoint fn calls `http_snapshot(pjson, p_os)`
with the ostream that carries the HTTP response body:

```cpp
auto http_fn = ([this, pjson_rp](std::string const &, Alist const &,
                                 std::ostream * p_os)
                { this->http_snapshot(pjson_rp, p_os); });
```

`HttpEndpointFn`'s `std::ostream *` is **deliberately staying** (decided
2026-08-06): it carries HTTP payload, not diagnostics, and an endpoint may
produce anything, not just JSON. So this ticket has to bridge the two — a
`FlatSink` over the response ostream at the http_snapshot boundary is the
obvious answer, and it is also where a future "pretty-print this JSON response"
switch would live.

## Questions to settle first

1. **Replace, or add?** `print(x, PpSink&)` alongside the ostream overloads,
   or a hard swap? With 9 pure-virtual subclasses, "add" means every subclass
   implements both, or a default that bridges one to the other. A hard swap
   is more churn at once but leaves one code path.
2. **Does prettified JSON actually get built in this ticket**, or does it just
   move the plumbing to PpSink so that prettifying becomes possible later? The
   motivation is the former; the safe increment is the latter. Note that
   without emitting `begin`/`split`/`end` around objects and arrays, moving to
   PpSink buys nothing observable — a FlatSink renders identically to the
   ostream it wraps. **The structure emission is the actual feature.**
3. **What about `jsonp_impl` / `operator<<`** (`PrintJson.hpp:100,115`)? Under
   the pattern used elsewhere these belong in a `printjson_ostream.hpp`, as the
   opt-in paved road for external users.

## Suggested sequencing

`print_json`'s 9 implementations are the risk. Consider doing the leaf
printers first behind a bridging default, so each conversion is verifiable
before the base class flips.

**Files:**
- `xo-printjson/include/xo/printjson/PrintJson.hpp` — `:31,39,44,55,100,115`
- `xo-printjson/include/xo/printjson/JsonPrinter.hpp` — `:32` and 9 subclasses
- `xo-printjson/src/printjson/PrintJson.cpp`
- `xo-reactor/include/xo/reactor/EventStore.hpp:67` — the ostream/PpSink seam
- `xo-printjson/utest/PrintJson.test.cpp` — asserts against `std::stringstream`

**Done when:**
- JSON serialisation writes into a `PpSink`
- flat output is byte-identical to today (pin this before adding structure)
- a prettified rendering is available, if question 2 says so

## Notes

Also blocked-adjacent: `xo-printjson` still uses legacy `iso8601` as an ostream
value (`PrintJson.cpp:384`), which has no ppsink equivalent of that shape. See
`.xo-backlog/xo-ppsink/issues/02-facility-gaps.md`. Closing that gap is a
prerequisite for xo-printjson's own indentlog migration, and this ticket would
subsume it.

Measure flat output before and after. Two separate incidents on this migration
came from predicting rendering rather than observing it.
