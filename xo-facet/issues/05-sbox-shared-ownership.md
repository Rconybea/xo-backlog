# 05 — `sbox<AFacet, DRepr>`: shared ownership of fomo objects

Status: open (design decided, not implemented)
Type: feature
Raised: 2026-09-28 (RC)

## Gap

fomo has borrowed handles (`obj`, `xo-facet/include/xo/facet/obj.hpp`) and
uniquely owning ones (`box`, `xo-facet/include/xo/facet/box.hpp`; and `abox`
for allocator-owned memory, `xo-alloc2/include/xo/alloc2/abox.hpp`), but no
shared ownership. Reference counting exists in xo (`ref::Refcount`, `rp<>`),
but it isn't used by fomo objects today (RC).

All three existing handles share one storage layout, `OObject<AFacet,DRepr>`:
a vtable pointer plus a data pointer.

```bash
grep -n "struct obj\|struct box\|struct abox" \
    xo-facet/include/xo/facet/obj.hpp xo-facet/include/xo/facet/box.hpp \
    xo-alloc2/include/xo/alloc2/abox.hpp
```

## Decided (RC, 2026-09-28)

- **An external count, not an intrusive one.** A heap control block holds the
  count; `DRepr` needs no refcount of its own, as with `std::shared_ptr`.
  Alternatives rejected:
  - refcounting as a facet, reached by rotation: needs the count inside
    `DRepr`;
  - the same with a cached ops table: a performance variant of the previous
    one;
  - an `ATop` vtable entry: refcounting isn't defined for every type, unlike
    `_typeseq` and `_drop` (`xo-facet/include/xo/facet/top/ATop.hpp`).
- **Heap only.** An arena frees in bulk, not per object, so a count reaching
  zero there reclaims nothing. Generalizing is possible but not worth it.
- **Handle layout `{iface, block*}`.** The block holds `{count, DRepr}`. Each
  handle keeps its own vtable pointer, so `to_facet<AOther>()` rotates while
  sharing the same block.
- **One allocation:** a factory in the style of `make_shared` puts the count
  and the `DRepr` in one heap block. Destruction is `_drop()` (already on
  `ATop`) and then freeing the block, so no vtable changes are needed.
- **Atomic count.**
- **Name: `sbox<AFacet, DRepr>`.**

## Consequences

- A call is `iface->method(block->data)`, one hop more than with `obj`.
- There's no way to turn a borrowed `obj` back into an `sbox`: a counted handle
  comes only from the factory. That's the accepted trade-off of an external
  count.

## Shape (to be settled when built)

- typed `sbox<A, D>` converts to type-erased `sbox<A>`, as `obj` and `box` do;
- `to_op()` gives a borrowed `obj`;
- copying adds a reference, moving transfers it, and destruction drops one.

## Related

- The planned `obj<>` → `fop<>` rename (mentioned in `reflectable2/spec.md`,
  `reflectable2/issues/02-establish-tdx-for-fomo.md`): keep the family's names
  consistent (`fop`, `box`, `abox`, `sbox`).
