# 01 — rewrite FacetRegistry::dump() against a PpSink

Status: open
Type: task

`xo::facet::FacetRegistry::dump()` still writes to an `std::ostream *` by hand:

```cpp
/* xo-facet/include/xo/facet/FacetRegistry.hpp */
void dump(std::ostream * p_out) {
    (*p_out) << std::endl;
    (*p_out) << "<FacetRegistry" << std::endl;
    for (auto & kv : registry_) {
        (*p_out) << "  [" << xo::pp::tostr(kv.first.first)
                 << "," << xo::pp::tostr(kv.first.second) << "]"
                 << " -> " << kv.second << std::endl;
    }
    (*p_out) << ">" << std::endl;
}
```

It should render into a `PpSink` instead, like every other printer in the tree.

**Deliberately not on a milestone.** RC's call, 2026-08-10: this does not block
`ppsink-migration`, and it is outside `ostream-containment`'s stated criterion
too — `dump()` declares no `operator<<` and includes no `*_ostream.hpp`, so it
is not among that milestone's 125 files. It is nonetheless the same kind of
thing, and would be natural to pick up while doing either.

## Why

- **It hand-rolls its own layout.** Indentation is two literal spaces, line
  breaks are `std::endl` — so the output cannot participate in an enclosing
  structure's line breaking, and a wide registry cannot fold. That is exactly
  what the ppsink model exists to fix.
- **It is the only place in xo-facet that touches an ostream.** Converting it
  would let `FacetRegistry.hpp` drop `<ostream>` from its transitive surface;
  the header is included by essentially the whole facet cluster.
- **Its `tostr()` calls are a stopgap I introduced on 2026-08-10**, replacing
  `<< kv.first.first` when `typeseq`'s inserter was removed
  (`.xo-backlog/xo-reflectutil/issues/01-typeseq-ostream.md`). They work and
  render identically, but a `tostr()` per key allocates a `std::string` per
  entry purely to hand it to an ostream. A PpSink renders the `typeseq` straight
  through `Prettifier<typeseq>` with no intermediate.

## Scope, measured 2026-08-10

**One caller**, so the signature is cheap to change:

```bash
grep -rn '\.dump(\|->dump(' --include=*.cpp --include=*.hpp xo-*/ | grep -v '/\.build/' | grep -i 'registry\|facet'
#   xo-gc/utest/Object2.test.cpp:117:  FacetRegistry::instance().dump(&std::cerr);
```

Whether to keep an ostream-taking overload for that caller, or move it to
`tostr()` / a `FlatSink`, is part of this ticket.

## Two things that need deciding, not just transcribing

**1. `kv.second` is a `const void *`.** The registry is
`DArenaHashMap<std::pair<typeseq,typeseq>, const void *, KeyHash>`
(`FacetRegistry.hpp:47,261`), and the value is streamed as a bare pointer today.
`Prettifier<const void *>` does not exist, so under a PpSink it would take the
`operator<<` fallback — i.e. the conversion would *introduce* a fallback use
rather than remove one. xo-ppsink has `hex.hpp` / `hex_ostream.hpp`; whether the
answer is a pointer Prettifier, `hex_view`, or `snprintf("%p")` as
`span_pp.hpp` does is open. **Check this before starting** — it is the part that
is not mechanical.

**2. The arity is a runtime loop, not a field pack.** So this wants
`sink.struct_open(...)` rather than `pretty_struct(...)`, and is a consumer for
`.xo-backlog/xo-ppsink/issues/06-dynamic-arity-struct-builder.md`. Worth
coordinating: if that ticket is going to change the builder's shape, doing this
one first means rewriting it twice.

## Suggested approach

Follow the phase-C discipline from
`.xo-backlog/xo-printable2/issues/01-aprintable-pretty-ppsink.md`, scaled down:
there is no second protocol to compare against here, but the same rule applies —
**observe the current output before changing it**, then pin it. The current
shape is a leading blank line, `<FacetRegistry`, one `  [a,b] -> ptr` line per
entry, then `>`; do not reproduce that from this description, capture it.

Note the registry's iteration order is a hash-map order and is not obviously
stable — check before pinning entry order, or sort for the test.

**Files:**
- `xo-facet/include/xo/facet/FacetRegistry.hpp` — `dump()`, and `registry_`'s
  key/value types at `:47` and `:261`
- `xo-gc/utest/Object2.test.cpp:117` — the only caller

**Done when:**
- `FacetRegistry` renders through a `PpSink`, with its output pinned by a test
- the `xo::pp::tostr()` stopgaps are gone
- `kv.second`'s rendering is a decision recorded here, not an accident of the
  `operator<<` fallback
