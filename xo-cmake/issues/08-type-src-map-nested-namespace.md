# 08 -- xo-type-src-map files `namespace a::b` types under `a::nested`

Status: done 2026-10-11 -- umbrella `b0fd1b93`
Type: bug
Raised: 2026-10-11, checking whether gcc/clang spelling breaks source links (`.xo-backlog/xo-websock/issues/12`)
Progress: grep -ho '"[A-Za-z_:]*::\(nested\|inline\)::[A-Za-z_]*"' .build/*/types.json | sort -u | wc -l

A type declared inside a C++17 nested namespace definition is keyed under
the wrong name: `namespace xo::fn { template <typename Tag> class
CallbackIdImpl ..}` (`xo-callback/include/xo/callback/CallbackId.hpp`) maps
as `xo::nested::CallbackIdImpl`, so `xo::fn::CallbackIdImpl<..>` -- what
reflection prints -- finds no source link.

## Diagnosis

clang's AST dump appends FLAGS after a namespace's name: `nested` for the
inner part of `namespace a::b`, `inline` for an inline namespace.  The
generator takes the last word of the line as the name
(`xo-cmake/cmake/xo_macros/xo-type-src-map.py`, the `NamespaceDecl` branch
of `parse_dump`: `re.search(r' (\w+)$', tail)`).

```bash
printf 'namespace xo::fn { struct A {}; }\nnamespace xo { inline namespace v1 { struct C {}; } }\nnamespace xo::a::inline b { struct D {}; }\n' > /tmp/ns.cpp
clang++ -std=c++20 -fsyntax-only -Xclang -ast-dump -Xclang -ast-dump-filter=xo /tmp/ns.cpp \
    | grep NamespaceDecl | sed -E 's/0x[0-9a-f]+//; s/<[^>]*>//'
#   `-NamespaceDecl   col:15 fn nested
#   `-NamespaceDecl   col:33 v1 inline
#     `-NamespaceDecl   col:25 b inline nested
```

(clang 19.1.0, 2026-10-11.)  The wrong keys, by subsystem:

```bash
cd "$(xo-sdlc --umbrella2)" && python3 -c '
import json,glob,collections
c=collections.Counter()
for f in glob.glob(".build/*/types.json"):
    for k in json.load(open(f)).get("types",{}):
        if "::nested::" in k or "::inline::" in k: c[f.split("/")[1]]+=1
print(sum(c.values()), dict(c))'
```

On 2026-10-11 every hit was `::nested::` (no xo type sits in an inline
namespace yet); xo-ppsink held most of them.  The unit test fixture
(`xo-cmake/utest/test_xo_type_src_map.py`) writes only `namespace xo {
namespace sub {`, which is why nothing caught it.

## Proposed fix

In the `NamespaceDecl` branch, drop trailing flag words before reading the
name -- the anonymous-namespace checks then see the line they expect:

```python
tail = re.sub(r'(?: (?:inline|nested))+$', '', line.rstrip())
```

Tests, in `TestParseDump` (hand-written dump text, no clang needed):
`fn nested` under `xo` -> `xo::fn::A`; `v1 inline` -> `xo::v1::C`;
`b inline nested` -> `xo::a::b::D`; and an anonymous inline namespace
(`inline namespace {`) still skipped -- its dump line is worth capturing
from clang rather than guessing.

**Done when:** the Progress command prints 0 after a rebuild, and
`xo::fn::CallbackIdImpl` has a source link in introspect.

## Resolution (2026-10-11, umbrella `b0fd1b93`)

As proposed: the `NamespaceDecl` branch drops trailing ` inline` /
` nested` before reading the name.  Four `TestParseDump` cases, dump lines
captured from clang -- `xo::fn::A`, `xo::v1::C`, `xo::a::b::D`, and an
anonymous inline namespace skipped -- each failed first with the bug's key
(`xo::nested::A`, `xo::inline::C`, `xo::nested::nested::D`,
`xo::inline::E`: so the anonymous one was mis-keyed too, not skipped).

**Two existing fixtures were the bug's own output.**  The regression tests
for "file advances on a non-decl line" and "a path inside a type string"
used a namespace literally named `nested` (`xo::nested::begin`), written
from the generator's mis-keyed map.  No xo namespace has that name
(`grep -rn 'namespace .*nested' --include=*.hpp`: none); the real type is
`xo::pp::begin` (`xo-ppsink/include/xo/ppsink/pretty.hpp`, `namespace
xo::pp {`).  Rewritten in the real dump shape (`xo`, then `pp nested`),
same intent; a comment in each keeps the old reading.

Done-when checked after rebuilding `.build` (every `types.json`
regenerates -- the script is a dependency): Progress command 0; key count
unchanged (renamed, not lost); `xo::fn::CallbackIdImpl` maps to
`xo-callback/include/xo/callback/CallbackId.hpp:22`, the key the page
derives from `xo::fn::CallbackIdImpl<xo::fn::CallbackId_tag>`.  Not
clicked through in a browser.

