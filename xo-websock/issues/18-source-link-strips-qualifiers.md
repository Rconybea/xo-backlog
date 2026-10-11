# 18 -- introspect: no source link for a `const T&` or `T*` member

Status: diagnosed
Type: bug
Raised: 2026-10-11, checking whether gcc/clang spelling breaks source links (`issues/12`)

`source_of()` (`xo-websock/example/introspect/mount-origin/introspect.js`)
strips only template arguments (`t.replace(/<.*$/, "")`) before looking a
canonical type name up in the type -> source map.  A member's DECLARED type
keeps its cv-qualifiers, reference and pointer, so it misses even when the
type is mapped:

| member's `_canonical_type_` | looked up as | in map as |
|---|---|---|
| `const xo::web::UrlRouter&` (WsSessionRouter `url_router_`) | `const xo::web::UrlRouter&` | `xo::web::UrlRouter` |
| `xo::web::WebserverImpl*` | `xo::web::WebserverImpl*` | `xo::web::WebserverImpl` |

Measured: every `_canonical_type_` in the golden snapshot, stripped as the
page strips it, against every generated map --

```bash
cd "$(xo-sdlc --umbrella2)" && python3 - <<'EOF'
import json,glob,re
keys={k for f in glob.glob('.build/*/types.json') for k in json.load(open(f)).get('types',{})}
g=json.load(open('xo-websock/utest/golden/server-snapshot.json')); ts=set()
def w(v):
    if isinstance(v,dict): [ts.add(x) if k=='_canonical_type_' else w(x) for k,x in v.items()]
    elif isinstance(v,list): [w(x) for x in v]
w(g)
for t in sorted(ts):
    k=re.sub(r'<.*$','',t)
    if k not in keys: print('MISS', t[:100])
EOF
```

On 2026-10-11, 23 of 42 missed.  Besides the two above, the misses were all
out of scope by design -- builtins, `std::`, `Json::CharReader` (external),
anonymous namespaces (skipped by the generator) -- and one generator bug,
`xo::fn::CallbackIdImpl` (`.xo-backlog/xo-cmake/issues/08`).  **None was a
gcc/clang spelling difference:** keys are bare qualified names, and the
spellings differ inside template arguments, which both sides drop.

## Proposed fix

In `source_of()`, strip leading cv-qualifiers and trailing `*` / `&` / `&&`
/ cv before the template arguments:

```js
const bare = t.replace(/^(?:(?:const|volatile)\s+)+/, "")
              .replace(/(?:\s*(?:\*|&&?|const|volatile))+$/, "")
              .replace(/<.*$/, "");
const loc = src.types[bare];
```

Links a pointer or reference member to its pointee's definition -- the
useful target.  Test, in `rows.mjs` (it runs with `--src-tree`): with the
map loaded, `source_of("const xo::web::UrlRouter&")` and
`source_of("xo::web::WebserverImpl*")` are non-null and name the same file
as `source_of("xo::web::UrlRouter")` / `("xo::web::WebserverImpl")`.

**Done when:** those two members link to their source, and the browser
tests pass.
