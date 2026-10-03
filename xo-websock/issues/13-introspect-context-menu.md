# 13 — introspect: a context menu on each box

Status: open -- increment 1 done (umbrella `5ebd6ac6`); refs / expand to come
Type: feature / example
Raised: 2026-09-29 (RC)

Part of making the introspect page (issue 10) interactive, and more faithful to
the C++ representation. Today a box's only action is left-click, which opens
the source of its type (issue 12, step 4c):

```bash
grep -n 'g.on("click"' xo-websock/example/introspect/mount-origin/introspect.js
```

## Mechanism

- The browser fires a `contextmenu` event; a page that calls
  `preventDefault()` on it suppresses the browser's own menu and can show its
  own. Common practice (Google Docs, Figma, VS Code web, Gmail).
- The same event fires for the keyboard Menu key / Shift+F10 and for a long
  press on touch screens, so handling `contextmenu` (not raw mouse buttons)
  covers those too.
- In the page: `node.on("contextmenu", (ev, d) => { ev.preventDefault();
  show_menu(ev.pageX, ev.pageY, d); })`. The menu as an absolutely positioned
  HTML `<div>` (not SVG: text, hover and focus come free); closes on a click
  elsewhere, Escape, resize or window blur -- NOT on scroll (see increment 1).
  No library.

## Coexisting with the browser's menu

- Take over right-click ONLY on boxes; blank canvas, the raw-json panel and
  links keep the browser's menu.
- Firefox: Shift+right-click always shows the browser's menu, whatever the
  page does. Chrome has no built-in bypass; apps sometimes show the browser's
  menu on a second right-click while theirs is open, or skip theirs when
  Shift is held.
- Right-click menus are invisible until known: consider a `⋮` button on
  hover for discoverability.

## Open

- **What left-click does** once there is a menu. One split: left-click selects
  or expands the box (e.g. its members); "open source" becomes a menu item.
- **Menu contents.** Candidates:
  - open source (the type's definition; perhaps also where its printer is);
  - show this object's json alone;
  - highlight what holds this object and what it holds (its refs);
  - expand to show its members, drawn as the C++ layout (by value, `rp<>`,
    borrowed pointer, container slot) -- see issue 10's faithfulness notes;
  - copy its id or type name.

## Done when

- right-clicking a box opens the page's menu with the agreed items; elsewhere
  the browser's menu is unchanged
- the menu opens from the keyboard (Menu key / Shift+F10) on a focused box
- Escape, a click elsewhere, resize or leaving the window closes it
- left-click behaves as decided

## Increment 1, 2026-09-29 -- umbrella `5ebd6ac6`

RC: "go ahead". Chosen to leave both open decisions settable later: left-click
UNCHANGED (opens source) until "expand" gives it something else to do; menu
items that need nothing new: Open source, Show JSON, Copy type name, Copy id.
Deferred: highlight refs, expand members.

- `introspect.js`: each box keeps its object (`obj`); boxes are focusable
  (`tabindex=0`, a dashed border on focus); `contextmenu` on a box ->
  `show_menu` at the pointer, or below the box when the event has no pointer
  position (keyboard). Header = the type; an item that cannot act is disabled
  with its reason as tooltip (e.g. "no link provider"); focus starts on the
  first enabled item; ArrowUp/Down move; Escape closes and refocuses the box.
  "Show JSON" shows the object alone below the graph (new `#detail`),
  scrolled into view. Copy uses the clipboard api where the page is a secure
  context (https, localhost), else a hidden-textarea fallback -- viewing from
  another host over http is not secure.
- `index.html`: `#ctxmenu`, `#detail`, their styles.
- Found while testing: closing on SCROLL closed menus opened while the page
  scrolled -- "Show JSON" scrolls smoothly, focusing a box can scroll it into
  view. The menu is in page coordinates and scrolls with its box, so scroll no
  longer closes it.

Checked in headless chrome over the devtools protocol (real right-clicks,
real key events; `cdp_menu.mjs`, a scratch script): menu on the server and
an endpoint; header is the type; Open source disabled without a link
provider, enabled and focused first with `--src-tree`; arrows; Show JSON
fills `#detail` and closes the menu; Escape and a click elsewhere close it;
blank svg: `defaultPrevented` false, the browser's menu untouched; a
keyboard-style contextmenu (no pointer position) opens it at the box, and
Escape returns focus to the box. NOT checked: a real Menu key / Shift+F10
(CDP key events do not make headless chrome fire contextmenu); clipboard
contents; the look -- for RC in a browser.

## Expand members -- design (RC, 2026-09-30)

- Members come from the PRINTERS, opt-in per member (cycles stay under the
  printer's control, and the useful level of abstraction is not predictable):
  `"_members_": [{"_name_": <C++ member name>, "_type_": <declared type>,
  "_value_": <printed as usual: scalar, object, {"ref": id}, null>}]`. The
  printer's existing keys (`id`, `refcount`, `endpoints`, ...) stay: several
  are not members. Kind (rp / pointer / owned / container / value) is derived
  by the page from `_type_`.
- Type information from xo-reflect: every type in scope is reflected.
- Generic printer for reflected structs: (c) unchanged for now -- `_members_`
  only from hand-written printers -- then migrate to (b), `_members_` only.
- Rejected for now: a static member layout from clang's FieldDecls (the map
  generator could record them), and a printer-free automatic layout.

## Expand members, step 0, 2026-10-01 -- reflect xo-websock's types

Umbrella `4edde25e`. At that commit: full build clean, ctest 49/49, sweep
73/73 build, 47 utest ok + 26 without.

- `WebsockAppcx(cfg, reflect_appcx, printjson_appcx)` (template ctor takes
  both from `deps`); calls `websock_reflect_types(reflect_appcx.type_table())`
  (new `websock_reflect.{hpp,cpp}`) before installing printers. pywebsock's
  `configure(config, reflect_appcx, printjson_appcx)` follows (imports
  `xo.reflect`, as pyprintjson does).
- `static void reflect_self(reflect::TypeDescrTable *)` on each public class,
  `TypeDescrTable` forward-declared in the header; the `.cpp` reflects its
  implementation types too, and only those (RC): `Webserver` (+
  `WebserverImpl`, `WebsocketSessionRecd`, `WsSessionSender<WebserverImpl>`,
  `WsSessionTable<WebsocketSessionRecd>`); `WebserverConfig` -- defined
  inline in `Webserver.hpp`, not in `Webserver.cpp` -- has its own
  `reflect_self` in a new `WebserverConfig.cpp`, called directly by
  `websock_reflect_types`; `WebsocketSink` (+
  `WebsocketSinkImpl`), `WsSessionRouter` (calls
  `WsSessionRouter::Subscription::reflect_self`), `DynamicEndpoint`,
  `UrlRouter`. No members yet: added as printers opt in. The table argument
  is unused: `StructReflector` registers in the process-wide table (RC: ok).
- xo-reflect `StructReflector`: `have_to_self_tp` also for
  `SelfTaggingDisplayable` (RC), so a base pointer reflects as its actual
  type. Test `struct-reflect-self-tagging-displayable-most-derived` -- failed
  before.
- Consequence (RC chose (c)): a `Webserver*` now reflects as `WebserverImpl`,
  for which no printer existed, so the generic struct printer printed it.
  The Webserver printer moved to `Webserver.cpp`, keyed on `WebserverImpl`
  ("we're not really adding a printer: there is no use case for looking up a
  printer on an interface"); `_type_` is `type_name<WebserverImpl>()` -- the
  `self_tp()` + `const_cast` route is gone. `WebsocketSink` is plain
  `Displayable`, so its printer is unaffected.
- Test `websock-types-are-reflected`: all 12 types complete structs, and a
  `Webserver*` resolves to `WebserverImpl` -- fails with the
  `websock_reflect_types` call removed. ctest 49/49; headless-chrome page
  test passes (server box from the moved printer); sweep 73/73 build, 47
  utest ok + 26 without.

Next: a `JsonMembers` helper in xo-printjson, and `_members_` on the server's
printer; then expand on the page.

## Expand members, steps 1-2, 2026-10-02 -- `JsonMembers`; the server's members

Umbrella `6606d613` (JsonMembers + the server's members), `b3e525ae`
(PrintJson reflected). Decided (RC): an unprintable member is an
error ENTRY, not a throw; `member_as<Declared>` tried; explicit `end()`;
`JsonPrinter_Webserver` a friend of `WebserverImpl`.

- xo-printjson `JsonMembers` (`JsonMembers.{hpp,cpp}`): ctor writes
  `, "_members_": [`; `member(name, value)` / `member_as<Declared>(name,
  value)` write `{"_name_", "_type_": <declared, type_name<>>, "_value_":
  <PrintJson::print_aux>}`; `end()` writes `]`, dtor asserts it was called.
  A member is printable iff its TARGET type -- unwrapping `rp<>`, `T*`,
  `std::vector<>` at compile time, since reflection cannot report a
  pointer's pointee -- has a json printer or is a complete reflected struct;
  else `{"_name_", "_type_", "_error_": "type not reflected: X"}`.
  `PrintJson::has_printer(td)` added. Tests (`JsonMembers.test.cpp`, 5): empty;
  scalars and strings; reflected struct by value, pointer and vector; an
  unreflected struct by value and pointer -> error entries, the rest still
  printed; `member_as<std::atomic<std::int32_t>>`.
- `WebserverImpl`'s printer adds `_members_`: `ws_config_`,
  `listen_port_` (`member_as<std::atomic<std::int32_t>>`), `state_`
  (`member_as<Runstate>`, its name), `pjson_`, `url_router_`,
  `session_table_`. Friendship: `friend JsonPrinter_Webserver;`, the printer
  forward-declared in the anonymous namespace first -- `friend class
  JsonPrinter_Webserver;` does not look into the anonymous namespace and
  befriended a new `xo::web::JsonPrinter_Webserver` (compile errors showed it).
- Live (introspect): `ws_config_`, `url_router_`, `session_table_` print as
  their reflected structs (no members yet); `listen_port_` `std::atomic<int>`
  7689; `state_` `xo::web::Runstate` "running"; `pjson_` -> `"_error_":
  "type not reflected: xo::json::PrintJson"` -- the omission check working.
  `webserver-prints-as-json` pins the names, types, values-or-error.
  ctest 49/49; sweep 73/73 build, 47 utest ok + 26 without.

Then (RC: "yes, reflect PrintJson"): `PrintJson::reflect_self(table)` (no
members), called by `PrintJsonAppcx`'s ctor -- which now uses the
`ReflectAppcx` it was already handed. `pjson_` prints. The server-json test
now walks the whole output and requires no `_error_` anywhere -- fails with
the `reflect_self` call removed ("pjson_: type not reflected:
xo::json::PrintJson"). ctest 49/49; sweep 73/73 build, 47 utest ok + 26
without.

Next: the page's "Expand".

## Expand on the page -- plan (RC, 2026-10-02)

- (A): an expanded box grows a compartment of member rows, and the layout
  reflows. Needs a FULLY AUTOMATIC layout -- which a later goal needs anyway
  (toggling which parts of the object graph are shown at all). Absorbs issue
  10's layout rework.
- Engine: ELK (elkjs, layered; variable node sizes, nesting, ports), from
  jsdelivr alongside d3 (not on cdnjs).
- Member tags: xo-reflect's metatype of the declared type.
- Left-click toggles expand; "Open source" stays in the menu.
- Steps: (1) `_metatype_` on members; (2) ELK layout replacing the
  hand-placed columns, same boxes and edges; (3) expand: member rows, nested
  objects expand in turn, `{"ref"}` values as edges from their rows, declared
  type links to source; (4) later: show/hide parts of the graph.

## Expand on the page, step 1, 2026-10-02 -- `_metatype_`

Umbrella `218cd240`. `JsonMembers` writes `"_metatype_"` after
`_type_` on every entry, error entries included: `metatype2str` of the
DECLARED type's `TypeDescr` (`Reflect::require<Declared>()`). NB `rp<T>` and
`T*` are both `pointer`; `std::atomic<int>` and an enum (`Runstate`) are
`atomic`. Tests: `JsonMembers.test.cpp` expectations carry it (failed before);
`webserver-prints-as-json` checks the server's six: struct, atomic, atomic,
pointer, struct, struct. ctest 49/49; sweep 73/73 build, 47 utest ok + 26
without.

## Expand on the page, step 2, 2026-10-02 -- automatic layout (ELK)

Umbrella `20f895c4`. The introspect page only (`introspect.js`,
`index.html`).

- `layout()` now builds the object graph only -- nodes, and edges with a
  kind: `link` (server -> endpoints, sessions), `owns` (session -> sender,
  subscriptions), `uses` (subscription -> stream endpoint), `holds` (ticker ->
  subscription); refcount accounting unchanged. No coordinates, no column
  headings.
- `draw()` renders the boxes, measures them, hands ELK (elkjs 0.12.0,
  `elk.bundled.js` from jsdelivr -- not on cdnjs -- beside d3) a graph with
  their sizes: `layered`, direction DOWN, ORTHOGONAL edge routing, spacing
  leaving room for the refcount badges. Boxes are placed and edges drawn as
  ELK routed them, one `path.edge.<kind>` each. Asynchronous: a newer draw
  supersedes one still laying out (`draw_seq`).
- Checked in headless chrome (two extra websocket clients: 3 sessions, 4
  subscriptions, a ticker hold): 16 boxes, 20 edges, 0 overlapping boxes;
  after two back-to-back Refreshes the same, no duplicates; the context-menu
  test (`cdp_menu.mjs`) passes unchanged. Screenshot reviewed: ownership flows
  down from the server; uses / holds edges drawn in their styles.
- Seen, not changed: (1) wider than the window with several sessions (about
  1420px) -- the page scrolls, as before; scale-to-fit possible later. (2)
  `/introspect`'s badge red (4, 3 expected) -- not the layout (accounting
  unchanged): likely the server holding the endpoint while it runs the very
  `receive` that takes the snapshot. Worth confirming.

## Expand on the page, step 3, 2026-10-02 -- expand

Umbrella `84d03b1a`. The introspect page only.

- A box whose object has `_members_` is expandable (`.expandable`, a ▸/▾
  before its label): left-click, Enter, or the menu's new first item
  "Expand"/"Collapse" toggles it. Left-click no longer opens source (RC);
  "Open source" stays in the menu. Open boxes and members are remembered
  across refreshes (`expanded`: box ids and member paths).
- An open box grows a row per member: `name: Type [metatype] = value`. Type
  shortened (namespaces dropped; the full name, and file:line if mapped, as
  its tooltip); clicking it opens its source. Value: a scalar inline (JSON,
  cut at 40 chars); an object by its short `_name_`, and if it has
  `_members_` of its own it toggles open in place, indented (▸/▾); a vector
  `[n]`; an `_error_` dimmed with ⚠; `{"ref": id}` -> an edge (kind `member`)
  from an ELK port on the box's right side at that row to the referenced
  object's box. Box size from the measured rows; ELK reflows.
- Checked in headless chrome (`expand.mjs`, scratch): the server expandable,
  an endpoint (no `_members_`) not; a real left-click opens it: 6 rows
  (`listen_port_: atomic<int> [atomic] = <port>`), box 40 -> 156px, no
  overlaps; a CONSTRUCTED snapshot (no live object has a ref member or nested
  members yet) adds a ref member -> one member edge, starting on the box's
  right edge at its row and ending on the referenced box -- and a nested
  member: closed, then opened in place by clicking its row (the box stays
  open); Enter closes the box. The context-menu test passes with Expand
  first. Screenshot reviewed.
- Seen: a right-side port to a box below-left loops around (correct, not
  pretty) -- revisit once real ref members exist.

## Members for WebsocketSessionRecd, 2026-10-02

Uncommitted, awaiting RC's review. Decided (RC): a ref member type
(`member_ref`); `outbound_q_` shown as its size (2b), "better than not
mentioned"; skip `mutex_`.

- xo-printjson: `JsonMembers::member_ref<Declared>(name, void const * p)` ->
  `"_value_": {"ref": json_id(p)}` or null -- for an object printed in full
  elsewhere (printing it here would repeat it, or recurse). `json::json_id(p)`
  (the address, as ostream writes it) now lives in xo-printjson;
  xo-websock's `web::json_id` calls it, so ids and refs cannot drift. Test
  `json-members-member-ref` (ref and null).
- The session printer (`JsonPrinter_WsSession`, friend of
  `WebsocketSessionRecd`; forward-declared in the anonymous namespace): after
  its keys, `_members_`: `output_buf_` (`OutputBuffer*`; `OutputBuffer` now
  reflected in `Webserver::reflect_self`), `sender_`
  (`member_ref<rp<WsSessionSenderImpl>>` -- the sender is printed in full
  under "sender"), `router_` (value, a `WsSessionRouter` struct, no members
  yet), `outbound_q_` (`member_as<std::deque<std::string>>`, "<n> queued":
  xo-reflect has no deque). `output_buf_` and the queue size read together
  under the session's mutex -- held only briefly elsewhere (send_text,
  lws_write_pending, unsubscribe_all), never while printing.
- Page: `short_type` also drops anonymous-namespace qualifiers, folds `> >`
  to `>>`, and shows `basic_string<char>` as `string` (display only; the
  tooltip keeps the full name).
- Tests: live `live-sessions-lists-each-connection` checks each session's
  four members, `sender_`'s ref == that session's sender id, and no
  `_error_` anywhere in the snapshot. Headless chrome (`session_expand.mjs`,
  scratch): a session box expands to 4 rows; `sender_` = → draws ONE member
  edge, ending at that session's own sender box -- the first live ref
  member; menu and expand tests still pass. Screenshot reviewed. ctest
  49/49; sweep 73/73 build, 47 utest ok + 26 without.
