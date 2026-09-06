# 04 — xo-pyprintable2: bind_printable

Status: open
Type: feature
Milestone: pyobject2

New module mirroring `xo-printable2`, shipping one header: a binder adding the
`APrintable` methods to any pybind class whose repr implements the facet.

```cpp
template <typename DRepr, typename PyCls>
void bind_printable(PyCls & cls) {
    cls.def("pretty", [](const H<DRepr> & h, PpSink & sink) {
        h.template _native_as<APrintable>().pretty(sink); });
    cls.def("__repr__", [](const H<DRepr> & h) {
        return h.template _native_as<APrintable>().display_string(); });
}
```

`H<DRepr>` is `DObjectHandle<ATop,DRepr>` and each binder recovers its own facet
(ticket 08). The binder needs
`IPrintable_DRepr` in the consuming TU; `OObject`'s `has_facet_impl`
static_assert reports a missing include as a compile error rather than a runtime
null (`sed -n '60,66p' xo-facet/include/xo/facet/OObject.hpp`).

The module has no python classes of its own — it exists so the binder sits with
the subsystem owning the facet, matching the 1:1 convention of `xo-pyfacet` /
`xo-pyindentlog2` / `xo-pyreactor2`.

**Done when:**
- a pybind class over any `DRepr` implementing `APrintable` gains working
  `pretty` and `__repr__` from one line of module init
