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

Umbrella `4d1f6ff9`. Decided (RC): a ref member type
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

## Members for WsSessionRouter, 2026-10-02

Umbrella `30ae2734`. Decided (RC): a C++ reference reported as
metatype `pointer`; `subscription_v_` as an array of refs.

- xo-printjson `JsonMembers`: `member_ref<Declared>` accepts a reference
  type (`metatype_of<T>`: no xo-reflect metatype for references -> pointer);
  new `member_refs<Declared>(name, std::vector<void const *>)` -> an array,
  each `{"ref": id}` or null (a released slot; positions kept). Tests (8):
  a reference member; refs with a null slot.
- `JsonPrinter_WsSessionRouter` (new; `xo::web`, not the anonymous namespace
  -- the header befriends it by name): `{_name_, _type_, id, _members_}`:
  `url_router_` (`member_ref<UrlRouter const &>`), `sender_`
  (`member_ref<rp<WsSender>>`, by `dynamic_cast<void const *>` -- the
  most-derived address, the id the sender's own printer writes), `pjson_`,
  `readjson_` (`member_as<unique_ptr<Json::CharReader>>`, "set"/"null"),
  `subscription_v_` (`member_refs`, read under the router's mutex). Skips
  `mutex_`. Before this the router printed as a member-less generic struct.
- Page: an array of objects / refs expands into `[i]` element rows (no type
  or tag of their own); a ref to an object drawn as no box reads
  "→ (not drawn)" -- e.g. `url_router_`, the server's, printed inside the
  server.
- Tests: live `live-sessions-lists-each-connection` -- each session's router
  members; `sender_` ref == the session's sender id (most-derived address
  matches); `subscription_v_`'s refs == the ids of the subscriptions printed
  under the session. Headless chrome (`router_expand.mjs`, scratch): session 1
  -> `router_` opens in place (the first live nested expand) ->
  `subscription_v_` -> `[0] = →`; member edges end at session 1's sender
  (twice: the session's and the router's `sender_`) and its subscription;
  `url_router_` "→ (not drawn)", no edge; no overlaps. The session, menu and
  expand tests still pass. Screenshot reviewed. ctest 49/49; sweep 73/73
  build, 47 utest ok + 26 without.

## Members for Subscription, 2026-10-02

Umbrella `a2f82ef7`. `JsonPrinter_Subscription` (a struct with
public members: no friendship) adds `_members_`: `sub_id_`, `stream_name_`,
`endpoint_` (`member_ref<rp<DynamicEndpoint>>` -- printed in full in the
server's endpoints), `callback_id_` (`member_as<CallbackId>`, its number:
`CallbackId` is not reflected; gcc spells the alias
`CallbackIdImpl<CallbackId_tag>`), `sink_` (`member_ref<rp<WebsocketSink>>`
by most-derived address -- printed in full under "sink"; on the page folded
into the subscription box, so "→ (not drawn)").

Tests: `WsSessionRouter.test.cpp` (the subscription json case) checks the
five names and values, `endpoint_`'s ref == the printed endpoint ref,
`sink_`'s ref == the printed sink's id, no `_error_`. Headless chrome
(`sub_expand.mjs`, scratch): a subscription box expands to 5 rows; ONE member
edge, from `endpoint_` to the `/demo` endpoint box; `sink_` says "not
drawn"; the router, session, menu and expand tests still pass. ctest 49/49;
sweep 73/73 build, 47 utest ok + 26 without.

## Members for UrlRouter, 2026-10-02

Umbrella `896ba0e4`. Decided (RC): including (b), edges to
objects printed nested in a box.

- xo-printjson `JsonMembers::member_ref_map<Declared>(name, vector<pair<string,
  void const *>>)` -> a json object, key -> ref or null, keys in the order
  given. Test (9 cases). NB xo-reflect has no map metatype: a std::map /
  unordered_map member reports `atomic`.
- `JsonPrinter_UrlRouter` (new, `xo::web`, befriended by name in
  `UrlRouter.hpp`; registered via `provide_url_router_json_printers`): `{_name_,
  _type_, id, _members_}` -- NOW WITH AN ID (the generic struct printer gave
  none) -- `http_map_` and `stream_map_` as ref maps (stem -> endpoint,
  printed in full in the server's endpoint list), copied under the router's
  mutex, sorted by stem (an unordered_map has no stable order). Skips
  `mutex_`.
- Page: (a) a ref map expands into `["stem"] = →` rows, each an edge to its
  endpoint box. (b) `box_of_id`: an object printed NESTED in a box (found by
  walking each box's `_members_` values for objects with an id) is drawn by
  that box -- a ref to it gets an edge to the containing box; only refs to
  objects drawn nowhere read "→ (not drawn)". So a session router's
  `url_router_` -> the server box. A ref into its own box draws nothing.
  ELK: ownership edges (link, owns) get `elk.layered.priority.direction` 10,
  others 0 -- the first url_router_ edge (session -> server) otherwise put
  session 1 ABOVE the server.
- Tests: server-json -- `url_router_` has an id; each printed endpoint is in
  its map by stem, ref == its id; no extra entries. Live -- a session
  router's `url_router_` ref == the server's `url_router_` id. Headless chrome
  (`urlrouter_expand.mjs`, scratch): server -> url_router_ -> two maps {2}
  each -> stream_map_ rows `["/demo/"] = →`, `["/introspect"] = →`, edges to
  those endpoint boxes; session 1 -> router_ -> url_router_ "= →", an edge to
  the server box; the server stays above its sessions. `router_expand`
  updated to the new expectation; the other browser tests pass. Screenshots
  reviewed. ctest 49/49; sweep 73/73 build, 47 utest ok + 26 without.

## Members for WsSessionTable, 2026-10-02

Umbrella `c542269b`.

- `JsonPrinter_WsSessionTable<Recd>` (a template in `xo::web`, defined and
  registered in `Webserver.cpp` for `WsSessionTable<WebsocketSessionRecd>`;
  the table template befriends it -- `template <typename> friend class`, forward
  declared above it): `{_name_, _type_, id, _members_}`: `next_id_` (read
  directly: the public `next_id()` increments it), `session_map_` as a ref map
  session id -> the session (printed in full in "sessions"), in id order,
  copied under the table's mutex. Skips `mutex_`. (In the template, the
  chained `.member_ref_map<..>` call would need `.template`: two statements
  instead.)
- Page: `short_type` also drops the standard library's default template
  arguments (`default_delete`, `hash`, `equal_to`, `less`, `allocator`,
  `char_traits`, brackets matched) and any space before `>` -- gcc spelled
  the session map's full type, 5 lines wide.
- Tests: server-json -- the idle server's table: an id, `next_id_` 1, an empty
  `session_map_`. Live -- `session_map_` has every printed session by id, ref
  == its id; `next_id_` beyond the last session id. Headless chrome
  (`sesstable_expand.mjs`, scratch): server -> session_table_: `next_id_ = 3`,
  `session_map_: unordered_map<long unsigned int,
  unique_ptr<WebsocketSessionRecd>> = {2}` -> rows `["1"]`, `["2"]`, edges to
  the two session boxes; the six other browser tests pass. ctest 49/49;
  sweep 73/73 build, 47 utest ok + 26 without.

## Members for DynamicEndpoint, 2026-10-02

Umbrella `8ee4099e`.

- `JsonPrinter_DynamicEndpoint` moved out of `websock_json.cpp`'s anonymous
  namespace into `xo::web`, befriended by name in `DynamicEndpoint.hpp`. Its
  `_members_`: `kind_` (`member_as<EndpointKind>`, "http"/"stream"),
  `uri_pattern_`, `uri_regex_` (`member_as<std::regex>`, "<n> captures" --
  `mark_count()`), `var_v_` (the pattern's variable names), `http_handler_`,
  `subscribe_fn_`, `unsubscribe_fn_` (`member_as<..>`: "set"/"empty"),
  `receiver_` (`member_ref<rp<StreamReceiver>>`, by most-derived address:
  receivers are printed nowhere -> "→ (not drawn)", or null).
- Page: an array of scalars shows its contents inline (`var_v_ =
  ["name"]`, cut at 40 chars) instead of `[n]`.
- Tests: server-json -- `/status` (http) and `/fw/${id}` (stream): all eight
  names; kind, pattern, "0 captures" / "1 captures", var_v_ [] / ["id"],
  handler / subscribe presence, receiver null. Headless chrome
  (`endpoint_expand.mjs`, scratch): `/introspect` (has a receiver) ->
  `receiver_ = → (not drawn)`; `/hello/${name}` -> "1 captures", `var_v_ =
  ["name"]`, handler "set", receiver null. `expand.mjs`'s "not expandable"
  check moved from an endpoint to the ticker (endpoints now have members);
  the other browser tests pass. ctest 49/49; sweep 73/73 build, 47 utest ok
  + 26 without.

## Members for WsSessionSender, 2026-10-03

Umbrella `d7027397`. `JsonPrinter_WsSessionSender` moved out of
`Webserver.cpp`'s anonymous namespace into `xo::web`, befriended by name in
`WsSessionSender.hpp`. Its `_members_`: `target_` (`member_ref<WebserverImpl *>`:
the server, printed in full elsewhere -> an edge to the server box),
`session_id_`, `open_` (`member_as<std::atomic<bool>>`, its `load()`).

Tests: live `live-sessions-lists-each-connection` -- each sender: `target_`'s
ref == the server's id, `session_id_` == its session's, `open_` true.
Headless chrome (`sender_expand.mjs`, scratch): a sender box expands to three
rows; one member edge, `target_` -> the server box; the eight other browser
tests pass. ctest 49/49; sweep 73/73 build, 47 utest ok + 26 without.

Every xo-websock box type now has members: server, session (router nested),
sender, subscription, endpoint; plus the server's url router and session
table. Only the example's own ticker has none.

## Showing / hiding parts of the graph -- plan (RC, 2026-10-03)

- Collapse an ownership subtree into its owner (option 2). The displayed graph
  is always connected (to the server); edges into hidden boxes are DROPPED,
  not redirected. Rejected for now: hiding single boxes, filters by kind,
  neighbourhood focus.
- Default: only the Webserver box; it cannot be hidden.
- Two kinds of expand: a disclosure triangle beside a box shows/hides its
  CHILDREN (owned boxes); left-click opens its MEMBERS, as before.
- Layout: a `tree`-command-like layout (children indented from the owner's
  left edge, a spine with a branch to each; non-tree edges in a gutter on the
  right) -- as an ALTERNATIVE to ELK, selectable (RC: a tree may not suit
  other drawings). Smooth d3 transitions between layouts, for either; a
  force-directed layout judged a poor fit (circle collisions vs wide boxes,
  no ownership direction, unstable across refreshes).
- Steps: (1) drop the ticker box; (2) collapse/expand children (ELK);
  (3) transitions; (4) the tree layout as an alternative.

## Step 1, 2026-10-03 -- drop the ticker box

Umbrella `9c5d830e`. RC: (a) -- the box goes, the Ticker
stays (`/demo` keeps ticking).

- `introspect.cpp`: `IntrospectSnapshot` no longer holds the ticker; it
  carries `app_holds` -- the ids of the sinks the application holds (by
  most-derived address, the id the sink's printer writes), for refcount
  accounting only. `JsonPrinter_Ticker` and its registration removed.
- Page: no ticker box, no "holds" edges (code and style removed); the sink
  accounting counts `app_holds` -- without it every `/demo` sink's badge
  would be red, one hold unexplained.
- Checked in headless chrome (`noticker.mjs`, scratch): no ticker box, no
  holds edges, `app_holds` one id, the `/demo` subscription's badge 2 (slot
  + ticker) and NOT red; the nine other browser tests pass (`expand.mjs` now
  checks there is no ticker box). ctest 49/49; sweep 73/73 build, 47 utest ok
  + 26 without.

## Step 2, 2026-10-03 -- show / hide children

Umbrella `3a793767`. The introspect page only.

- A box's CHILDREN are the boxes it owns ("link" / "owns" edges). A ▸n / ▾
  left of a box with children shows / hides them -- also ArrowRight /
  ArrowLeft on a focused box, and the menu's "Show children (n)" / "Hide
  children". `children_open` (box ids) is kept across refreshes; default
  empty: ONLY the Webserver box, which is always shown. A box is drawn iff
  every owner above it shows its children; edges with a hidden end are
  DROPPED (RC). "Show all" / "Hide all" buttons beside Refresh.
- Refs: joined against EVERY box, shown or not: to a hidden box "→
  (hidden)" and no edge; drawn nowhere "→ (not drawn)".
- Members stay on left-click; the label's ▸/▾ (members) became a trailing
  " ⋯" while members are closed -- the triangle now means children.
- Checked in headless chrome (`children.mjs`, scratch, real clicks and key
  events): default only the server, toggle `▸6` (4 endpoints, 2 sessions),
  no edges; clicking it shows the 6, toggle `▾`, members not opened;
  ArrowRight on a session shows its sender and subscription (and the uses
  edge), ArrowLeft hides them; a session-map ref to a hidden session reads
  "→ (hidden)" with no edge, shown "→" with 2 member edges; the menu item;
  Hide all -> only the server, Show all -> all 11 boxes, no overlaps. The
  ten earlier browser tests now click "Show all" first (they waited for
  edges, of which the default view has none); the menu test's expectations
  follow the new "Hide children" item. Screenshot reviewed. ctest 49/49;
  sweep 73/73 build, 47 utest ok + 26 without.

## Step 2b, 2026-10-03 -- shown is per box

Umbrella `bfe2b441`. RC: "are we displaying a box" should be a
property of each box -- e.g. `/types` without the sessions, or vice versa.
The introspect page only (`introspect.js`, `index.html`).

- Replaces step 2's `children_open` (per OWNER) with `shown` (box ids), kept
  across refreshes. A box is DRAWN iff it is in `shown` or on the ownership
  path from the server to one that is -- so the drawing stays connected; the
  Webserver always. Hiding a box hides its descendants too. Edges with an
  undrawn end still dropped.
- Box triangle `▸k` (k = children not drawn) / `▾` (all drawn): shows all its
  children / hides them all; ArrowRight / ArrowLeft the same.
- Menu: "Hide" (disabled on the Webserver), and "Show ▸ <label>" per hidden
  child, beside "Show children (k)" / "Hide children".
- Ref rows (RC): a ref to a box other than the Webserver carries a `▸` / `▾`
  before the arrow -- e.g. the compact Webserver's
  `url_router_.stream_map_["/introspect"]`; `▸` shows that box (and the path
  to it), so its edge appears; `▾` hides it.
- Checked in headless chrome (`visibility.mjs`, scratch, replacing
  `children.mjs`; real clicks): default only the server, `▸6`; the server's
  Hide disabled; `stream_map_["/introspect"] = ▸→ (hidden)`, clicking the ▸
  draws exactly the server and `/introspect` with one member edge, row
  `▾→`, server toggle `▸5`; ▾ hides it again; showing `session:1:sub:0`
  draws server, session:1, the sub -- not the sender nor session:2; the
  session menu's "Show ▸ sender" and "Hide" (descendants go too); the box
  triangle both ways; `/types` alone, session:1 alone; Show all 11 boxes, no
  overlaps; Hide all. The ten other browser tests pass, ref-row expectations
  now `▾→`, the menu test's following the new "Hide" item. JS only: no
  ctest / sweep rerun.

## Member edges leave the bottom; hover pairs a row with its edge, 2026-10-03

Umbrella `90c18b4e`. The introspect page only (`introspect.js`,
`index.html`). RC: edges from the boxes' right side dogleg back to the left.

- Tried, in headless chrome, "Show all" with the server's maps, session 1
  and its `/demo/1` subscription open: EAST ports (before) 1802px wide, edges
  run right then back; WEST ports 1967px -- ELK moves the server right to
  route on its left, doglegs now rightward; SOUTH ports 1473px, edges flow
  down with the ownership edges. ELK's layered DOWN layout wants edges out
  of the bottom; any side port routes around the box. RC: keep SOUTH.
- A ref row's port now sits on the box's bottom edge, near its left, 10px
  apart in row order. An edge no longer starts at its row, so (RC) hovering
  a ref row lights its member edge (orange, 3px, raised above the others)
  and the row's name; hovering the edge lights its row.
- Checked in headless chrome: `expand.mjs` now checks the edge leaves the
  bottom edge near the left (was: the right edge at its row), and, with a
  real mouse move, that hovering the row lights its edge and only that row,
  moving away restores both, hovering the edge lights the row. The ten
  other browser tests pass unchanged. Screenshot of a hover reviewed. JS
  only: no ctest / sweep rerun.

## Ownership edges not drawn; member edges enter top-left, 2026-10-03

Umbrella `90c18b4e`. RC: a connection
drew twice -- grey ownership edge (from the box's corner: portless edges on
a FIXED_POS box) and purple member edge; "I don't know a reason to see the
ownership edges visually"; member edges should arrive offset from the
target's top-left corner, so a descending edge need not dogleg left.

- Ownership edges (link, owns) stay in the ELK graph, where they order the
  layers top to bottom (priority 10), but are not drawn. A box only owned,
  never referred to by an open row, now floats with no edge.
- A box a member edge arrives at gets one NORTH port, `in_port_x` (24px)
  from its left; every member edge into it targets that port.
- Checked in headless chrome: `expand.mjs` now also checks the member edge
  ends on the endpoint's top edge 24px from its left, and that no link /
  owns path is drawn; all 11 browser tests pass. Screenshot reviewed.

## Visibility from wanted edges, 2026-10-03

Umbrella `90c18b4e`. The introspect
page only. RC: state was missing for which edges to display -- collapsing
lost an edge but left its box floating; a box shown by the triangle or menu
got no edge once ownership edges stopped drawing. RC: collapsing a box should
hide what it connects to, unless visible through another edge; a box shown
without a ref row draws its edge as if the row were open.

- State is now `wanted`, a set of ref edges keyed by row key (replaces
  `shown`). Every ref anywhere in a box's members -- open or not -- is a
  showable edge (`all_refs`); an owned child no owner ref reaches would get
  its ownership edge as a fallback (none today).
- DRAWN: the Webserver plus what wanted edges reach from it. Every wanted
  edge between drawn boxes draws from its box's bottom, row open or not; so
  does a visible ref row's edge to a box drawn anyway. An edge's tooltip
  names its box and member path (e.g. `session_table_.session_map_["1"]`),
  since its row may be closed.
- Ref row ▸/▾ wants / unwants its edge. Box triangle and menu "Show ▸"
  want the owner's ref edge to the child. Menu "Hide" unwants every edge
  into the box. COLLAPSING A BOX unwants every edge out of it; what was
  reached only through those goes. A member row closing (e.g. stream_map_)
  does NOT (RC: its edges still visibly leave the open box). Show all wants
  every edge but those into the Webserver (always drawn; wanting one keeps
  nothing shown, and drew long back-edges); Hide all clears.
- Wanted edges out of a box no longer drawn are kept (RC: my discretion):
  show the box again and what hung off it returns.
- Checked in headless chrome: `visibility.mjs` rewritten -- among others,
  Hide session:1 drops its sender and sub, showing it again brings them
  back; the triangle on a COLLAPSED server draws 6 member edges, tooltips
  naming `session_map_["1"]` .. `stream_map_["/introspect"]`; closing
  stream_map_ keeps both endpoints and edges; collapsing the server drops
  everything it alone reached; collapsing a sub whose endpoint_ edge was
  wanted keeps /demo/ (the server's edge still wants it). The five
  expand tests that start from Show all now count only edges whose row is
  open; `expand.mjs`'s port bound allows the 7th port. All 11 pass.
  Screenshot of Show all reviewed.

## Node placement: network simplex, 2026-10-03

Umbrella `90c18b4e`. RC: the
Webserver box jumps away from the left margin when sub 0 opens.

- Cause: ELK's default node placement, Brandes-Koepf, computes four
  candidate placements (align with upper / lower neighbours, sweeping left /
  right) and keeps the narrowest. In RC's view (server and sub 0 open;
  /hello, session 1, sender, sub 0, /introspect shown; /introspect via
  stream_map_), opening sub 0 (176 -> 457px) left two candidates within a
  pixel of each other (786px); the one kept aligns the server with its edge
  to /introspect: server x 32 -> 244.
- Measured in that view, ELK options overridden in the tab: fixed LEFTDOWN
  65 -> 65; BALANCED 138 -> 154; NETWORK_SIMPLEX 65 -> 65 and the narrowest
  with sub 0 open (span 53..660 vs 53..839). RC: network simplex.
- `"elk.layered.nodePlacement.strategy": "NETWORK_SIMPLEX"` on the root.
  All 11 browser tests pass, no overlaps; the expanded view used for the
  edge screenshots is now 1338px wide (was 1804). Network simplex centres a
  parent over its children, so with many children shown the Webserver sits
  mid-canvas rather than at the left margin -- but stays put.

## Menu: a Hide ▸ / Show ▸ entry per child, 2026-10-03

Umbrella `6dc304f5`. The introspect page only (`introspect.js`).
RC: the menu offered "Show ▸ <child>" only for a child not drawn; add a
"Hide ▸ <child>" for a child that is.

- A box's menu now lists one entry per child, in ownership order: `Hide ▸
  <label>` if drawn, `Show ▸ <label>` if not (as before: wants the owner's
  edge to it).
- Hide ▸ <child> does what the child's own "Hide" does -- unwants every edge
  into it (RC: option (a)). Rejected (b), unwanting only this box's edge to
  it: the exact inverse of Show ▸, but when another wanted edge reaches the
  child, the item appears to do nothing.
- Checked in headless chrome (`visibility.mjs`): with both of session 1's
  children drawn, its menu has `Hide ▸ sender`, `Hide ▸ sub 0 · /demo/1`, no
  Show ▸; `Hide ▸ sender` drops the sender, keeps the sub, and the entry
  flips to `Show ▸ sender`; with /demo/ reached by the server's edge AND the
  sub's `endpoint_` edge, the server menu's `Hide ▸ /demo/${id}` still hides
  it, the sub stays. `cdp_menu.mjs` reworked for the longer server menu:
  checks the Hide ▸ entries, and that ArrowDown skips the disabled Hide and
  the disabled Open source. All 11 browser tests pass.

## Parallel member edges merge, 2026-10-03

Umbrella `6dc304f5`. The introspect page
only. RC: with every box drawn, session 1 drew two edges to its sender.

- Cause (headless chrome, Show all): two refs from session 1's box to the
  same sender -- `session:1/sender_` (rp<WsSessionSender<WebserverImpl>>,
  the record's) and `session:1/router_/sender_` (rp<WsSender>, the router,
  a struct member printed in the session's box); same id. Both wanted by
  Show all, both drawn. Accurate (the sender's badge 3: record, router,
  sink) but clutter.
- RC: merge parallel edges from the same box. `merge_parallel()`: one drawn
  member edge -- and one bottom port -- per (source box, target box), in
  order of its first member, carrying every member's row key and label.
  Wanted-ness stays per row: the line is drawn while any of its members is
  wanted or its row open. Tooltip lists them (`session 1 · sender_,
  router_.sender_`). Hovering the line lights every row it stands for;
  hovering a row lights the line and that row only.
- Checked in headless chrome: `router_expand.mjs` now expects ONE edge to
  the sender carrying both row keys and both labels in its tooltip, edge
  hover lighting both rows, row hover lighting the edge and that row only.
  The five tests filtering "edges whose row is open" read `row_keys`. All 11
  browser tests pass.

## Compact member rows: type on tooltip + row menu, 2026-10-03

Umbrella `870cd295`. The introspect page only (`introspect.js`,
`index.html`). RC: move a member's type name and metatype off the row, into
a tooltip / context menu on the member name, so expanded boxes get narrower
-- keeping a way to reach the type's definition.

- A row reads `name = value` (was `name: Type [metatype] = value`).
- Hovering the name: tooltip `name: Type  [metatype]`, the canonical type,
  `file:line` (or "no source location"), and "ctrl-click: open source"
  when linked.
- Ctrl-click / cmd-click the name (RC: yes): opens the type's source; does
  not toggle the row. (On macOS ctrl-click is a right-click: the row menu,
  which has Open source.)
- Right-click a row: a ROW menu, headed `name: Type` -- Open source, Copy
  type name (canonical), Expand / Collapse if it opens, Show ▸ / Hide ▸
  <target> on a ref row (wants / unwants its edge, as its ▸ / ▾).
  Disabled with a reason where they don't apply (element rows: "no declared
  type"). Right-click elsewhere on the box: the box menu, as before.
  `show_menu()` now takes a heading and items, shared by both menus.
- A "types" checkbox (RC: yes) beside Show all / Hide all puts types back
  inline. Off by default.
- Rows are not focusable, so the row menu is mouse-only; making them
  focusable would change keyboard movement through the graph -- not done.
- Webserver box (server open, url_router_, stream_map_, session_table_,
  session_map_ open): 266px compact vs 595px with types; the expanded view
  used for edge screenshots 1035px wide (was 1338).
- Checked in headless chrome: `rows.mjs` (new, scratch; server with
  `--src-tree`, `window.open` stubbed) -- compact rows and no type tspans;
  the tooltip text; ctrl-click and cmd-click open
  `/dyn/src/xo-websock/include/xo/websock/UrlRouter.hpp#L48` and leave the
  row open; the row menu's head and items, its Open source; an element
  row's menu (Open source disabled, Hide ▸ /introspect unwants the edge,
  flips to Show ▸); right-click on the box header still the box menu;
  types on -> `listen_port_: atomic<int> [atomic] = ..`, wider; off ->
  back. The 11 earlier browser tests tick "types" at load (they find rows by
  `name:` text) and pass unchanged.

## Rows align on " = " per sibling group, 2026-10-03

Umbrella: never committed as such -- superseded (next section) before `406133a6`. The introspect page only (`introspect.js`).
RC: align each `member = value` row on its `=`.

- Offered (a) one `=` column per box (any depth) or (b) one per sibling
  group (rows under the same parent). RC: (b), likely to look better.
- Each row records its `parent` path. After a box's rows are drawn, the x
  where each row's ` = ` starts is measured (`eq_x()`:
  `getStartPositionOfChar` at the count of the name tspans' own characters
  -- not their `<title>` text); a group's column is its largest; the ` = `
  tspan (class `meq`) gets that x. A box is now sized by each row's bbox
  (the moved ` = ` is not in `getComputedTextLength`).
- The "types" view aligns the same way, on the longer `name: Type
  [metatype]`; there one long type stretches its whole group (e.g.
  session_map_'s `unordered_map<long unsigned int,
  unique_ptr<WebsocketSessionRecd>>` pushes next_id_'s `= 3` far right).
- Checked in headless chrome: `rows.mjs` -- with server, url_router_,
  stream_map_, session_table_ open, every sibling group's ` = ` at one x,
  the columns differ between groups, the group's longest name runs straight
  into its ` = ` (gap < 1px). All 12 browser tests pass; screenshots of both
  views reviewed.

## " = " steps in with nesting: x0 + indent * depth, 2026-10-03

Umbrella `406133a6` (supersedes the per-sibling-group rule of the
previous section, never committed). RC: nested members' `=` should indent by
the same amount the names indent -- a row at nesting level n puts its `=` at
x0 + d*n, d the per-level text indent.

- One x0 per box: the least that clears every row's name, i.e. the max over
  rows of (unaligned ` = ` x - d * depth). Each ` = ` (tspan `meq`) gets
  x0 + d * depth. d is now a named constant, `row_indent` (14px), used for
  both the names' indent and the `=` step. Rows no longer carry `parent`.
- Per-sibling-group alignment (previous section) is gone: it let a nested
  group's `=` sit LEFT of its parent's (url_router_'s children at 583px vs
  the top level's 591px) -- a ragged column RC's rule removes.
- With server, url_router_, stream_map_, session_table_ open: `=` at 598 /
  612 / 626 px for depths 0 / 1 / 2; x0 is set by `["/introspect"]` (depth
  2); the Webserver box ~8px wider than per-group.
- Checked in headless chrome: `rows.mjs` -- every row's `=` at x0 + 14 *
  depth (spread 0.00px), rows at 3 depths, the tightest name-to-` = ` gap
  0.00 (x0 is the least that fits). All 12 browser tests pass; screenshot
  reviewed.

## Box menu button, 2026-10-03

Umbrella `dcfe96bd`. The introspect page only (`introspect.js`,
`index.html`). RC: left-clicking the meatballs should show the menu; put
them in a rounded square so they read as a UI element; left-click elsewhere
on the box still toggles it.

- The label's trailing `⋯` was an EXPAND hint (a box with members, while
  closed), not a menu. Now every box -- open or closed, with members or not
  -- has a menu button just after its label: a square (RC: equal x,
  y extent) holding three drawn dots (SVG circles: the `⋯` glyph is tiny in
  the monospace font). Left-click opens the box menu just below it and does
  not toggle the box; elsewhere on the box, left-click toggles as before;
  right-click and Menu / Shift+F10 unchanged.
- Looks: first a white-filled, outlined square -- RC: too busy. Then plain
  (transparent fill, which still takes the click) until hovered, hover a
  light-blue fill and outline. Now (RC) hover is a borderless rounded square,
  white, part-transparent, so it takes a hint of the box's hue. At 60%
  opacity RC found it barely visible; now 85%, and the square 21x21 (15%
  larger than 18).
- Considered three stacked lines (hamburger): conventionally a page / app
  menu; dots (meatballs / kebab) conventionally a per-item actions menu,
  which this is. RC: stay with dots.
- The "has members" hint is dropped (RC: option (a)): the pointer cursor
  shows only on boxes that open, and the menu's Expand is disabled, with a
  reason, on the rest.
- Box styles now select the box's own rect (`.node > rect`), so they don't
  restyle the button's.
- Checked in headless chrome: `cdp_menu.mjs` -- every box has a square
  button, the label no longer contains `⋯`, a real left-click on the button
  opens the box menu just below it and leaves the box as it was, a real
  left-click on the label toggles the box with no menu. Plain and hover
  screenshots reviewed. All 12 browser tests pass.

## Menu: children counts both ways; "Hide <label>", 2026-10-03

Umbrella `71cd5742`. The introspect page only (`introspect.js`).
RC (changing an earlier instruction): "Hide children" should carry a count
as "Show children" does, and a partly shown box should offer both -- e.g.
the Webserver with only /introspect drawn: "Show children (+5)" and "Hide
children (-1)". And "Hide" should read "Hide <the box's name>".

- `children_items()`: "Show children (+k)", k children not drawn, wants the
  edges to just those (`show_children()`); "Hide children (-m)", m drawn,
  hides just those (`hide_children()`). Each appears only when its count is
  non-zero, so all-hidden shows only Show, all-drawn only Hide, partly shown
  both. A box with no children keeps one disabled "Show children" ("owns no
  boxes"). The box triangle (`toggle_children()`) now calls the same two
  functions; its behaviour is unchanged.
- "Hide" -> `Hide ${label}`, e.g. "Hide session 1", "Hide Webserver :7680
  (running)" (still disabled on the Webserver). Labels keep the menu's
  sentence case ("children", not "Children").
- Checked in headless chrome (`visibility.mjs`): nothing drawn -> "Show
  children (+6)" only; RC's case -> both "(+5)" and "(-1)"; "Hide children
  (-1)" leaves the Webserver alone; "Show children (+5)" draws the other 5
  (7 boxes); then only "Hide children (-6)", which clears them; "Hide
  session 1". `cdp_menu.mjs` follows the new labels (server menu after Show
  all: "Hide children (-5)", "Hide Webserver :N (running)" disabled). All
  12 browser tests pass.

## Row values: refs read `▸ (→)`; structs just `▸`, 2026-10-03

Umbrella `10500227`. The introspect page only (`introspect.js`).
RC: rows whose value has no handy label (e.g. a Subscription's `endpoint_`
read `▾→`); the triangle looks bad with an arrow beside it.

- A ref row now reads `▾ (→)` / `▸ (→)` (target drawn / not); the
  `(hidden)` suffix is gone -- the triangle says it. A ref to the Webserver
  (no triangle) reads `(→)`; one no box draws, `(→ not drawn)`. The `(→)`
  marks the triangle as showing ANOTHER box, not rows in place.
- What a ref refers to is the value's tooltip (`ref_tooltip()`): "refers
  to <box label>" or "refers to an object printed inside <box label>",
  "(hidden)" if so, and the id. I offered naming the target in the row
  (`▾ (→ /demo/${id})`); RC: plain `(→)`.
- Struct rows that open: just `▸` / `▾` -- the type is on the name's
  tooltip and in the "types" view (was `▸ UrlRouter`). One that cannot open
  still shows its name (`ws_config_ = WebserverConfig`). Arrays / maps keep
  their counts (`▸ {2}`).
- Rejected markers: `(pointer)` clashes with the metatype word (a reference
  member is reported as pointer too); `(out)` vague.
- Checked in headless chrome: `rows.mjs` adds the ref row text and its
  tooltip ("refers to /introspect\n0x.."); every test's row-text
  expectations follow (`▾ (→)`, `(→ not drawn)`, `= (→)`, `= ▸`). All 12
  browser tests pass; screenshot reviewed -- the `→` glyph is small in the
  monospace font (as `⋯` was).

## Triangles 25% larger; hover squares inside boxes, 2026-10-03

Umbrella `c7bfc177`. The introspect page only (`introspect.js`,
`index.html`). RC: make the triangles ~25% larger, on a rounded square with
the menu button's colour policy -- the square only for triangles drawn
inside boxes.

- Three kinds, all larger: the children toggle beside a box (`text.kids`,
  13 -> 16px, no square); and in rows, the in-place toggle (struct / array /
  map) and the ref toggle (`▸ (→)`) -- each now its own tspan, class `tri`,
  at 125% of the row's font (12 -> 15px). The in-place triangle used to be
  part of the value string (`▸ {2}`); now `▸` + ` {2}`, so row text reads
  the same. A struct that opens has no value tspan at all (it was an empty
  one).
- `tri_buttons()`: behind each row triangle, a 16px rounded square (rx 3) --
  SVG text takes no background, so a rect placed from the triangle's
  measured bbox, after the `=` alignment. Plain (opacity 0) until the
  triangle or the square is hovered, then white at 85% -- the menu button's
  policy. Clicking the square does what clicking the triangle does
  (dispatches the click to it). Clicking elsewhere on a row still toggles
  it, as before.
- Checked in headless chrome: `rows.mjs` -- a square per triangle (6 in the
  opened Webserver box), triangles at 125% of the row font, squares square
  and plain; hovering a triangle shows its square at 0.85, leaving hides it;
  clicking stream_map_'s square closes / reopens it; a ref row's square
  unwants / wants its edge; the children toggle is 16px with no square. All
  12 browser tests pass; hover screenshot reviewed.

## Collapsing a box no longer hides what it shows, 2026-10-03

Umbrella `27ff8ddf`. The introspect page only (`introspect.js`).
RC: collapsing a box dropped its edges -- hiding what only it kept shown --
"at my instruction, but I think that was a mistake": collapsing should leave
the set of drawn boxes unchanged.

- Reverses one rule of "Visibility from wanted edges" above (RC then:
  "collapsing a box should also hide things it's connected to, unless those
  things would be visible through some other edge"). Why it was plausible:
  a collapsed box no longer shows the rows its edges come from, so the
  edges looked orphaned. Since then member edges leave from a box's bottom
  edge whether or not its rows are open, and carry their member path in a
  tooltip -- so a collapsed box's edges read fine.
- `toggle()` now only opens / closes; `drop_edges_from()` is gone.
  Collapsing is now symmetric with closing a member row, which never
  dropped edges.
- Checked in headless chrome (`visibility.mjs`, collapse section rewritten):
  collapsing the server keeps the same 5 boxes and the same member edges,
  the server's leaving its collapsed box, its edges still wanted; collapsing
  a sub keeps its `endpoint_` edge wanted, nothing drawn changes. All 12
  browser tests pass.

## A ref row's triangle follows its target box, 2026-10-03

Umbrella `27ff8ddf`. The introspect page only (`introspect.js`).
RC: session 1's `sender_` triangle did not toggle whether the sender is
drawn.

- Cause, reproduced in headless chrome after Show all + opening session 1:
  the session reaches its sender by `sender_` AND by `router_.sender_`
  (parallel edges, merged into one line). The row's triangle toggled only
  its own edge: clicking ▾ unwanted `session:1/sender_`; `router_.sender_`
  still wanted, so the sender stayed drawn (and the merged line with it)
  while the row read ▸. Not only parallel edges: any other edge into the
  target (another box's) does the same.
- Fix (RC: first option): the triangle follows the TARGET box -- ▾ while it
  is drawn, by any path; clicking ▾ hides it as menu "Hide ▸ <child>" does
  (unwants every edge into it; option (a) earlier); ▸ wants this row's
  edge, as before. `toggle_ref()`; the row menu's Show ▸ / Hide ▸ <target>
  use it too. Rejected: toggling the whole merged line (all this box's
  parallel edges) -- fixes sender_ but not a target another box also
  reaches.
- Consequence: a ref row can no longer ADD a second wanted edge into a
  target already drawn (its ▾ hides). `visibility.mjs`'s collapse section
  used that as setup; it now sets the edge directly.
- Checked in headless chrome: `visibility.mjs` -- after Show all, sender_
  reads ▾; clicking it hides the sender despite router_.sender_, and it
  reads ▸; clicking ▸ shows it again. All 12 browser tests pass.

## Names right-justified against the " = ", 2026-10-03

Umbrella `0a931674`. The introspect page only (`introspect.js`).
RC: right-justify member names relative to the anchored `=`.

- The `=` column is unchanged: x0 + row_indent * depth, x0 the least that
  clears every name. A row's text now STARTS at its column less the width
  of what precedes its ` = ` (measured unaligned: the name; with "types",
  `name: Type [metatype]`), so every name ends at its `=`. Box width
  unchanged; no name crosses its depth's indent, and the widest sits at it.
- Nesting now reads from the `=` staircase rather than the names' left
  edges, which are ragged (shown to RC before: a matter of taste).
- Checked in headless chrome: `rows.mjs` -- every name ends at its `=`
  (gap -0.78..0.00 px), none starts left of 12 + 14 * depth, the widest at
  it. All 12 browser tests pass (visibility.mjs once printed nothing in the
  batch run; alone, 48 checks ok -- not reproduced). Screenshot reviewed.

## No "=" in member rows, 2026-10-03

Umbrella: never committed as such -- superseded (next section) before `0a931674`. RC: drop the `=`
entirely -- bold name vs normal value, and the alignment, cue enough.

- The ` = ` tspan (class `meq`) stays as the alignment anchor but holds
  `row_sep` -- two spaces -- so a value never touches a bold name. Rows read
  `listen_port_  8790`, `url_router_  ▾`, `["/introspect"]  ▾ (→)`; with
  "types", `listen_port_: atomic<int> [atomic]  8790` (no `=` there either).
- Tests: 53 row-text expectations in the browser tests changed ` = ` ->
  two spaces (string literals only; one template literal by hand). All 12
  pass; screenshot reviewed.

## Separator ":" (types view keeps "="), 2026-10-03

Umbrella `0a931674` (with the right-justification section). RC, on the
"=" -less rows: "doesn't look as good as I expected. Let's try : instead of
=".

- Compact view: `row_sep` is now `": "`, right after the right-justified
  name: `listen_port_: 8830`, `url_router_: ▾`, `["/introspect"]: ▾ (→)`.
- "types" view keeps ` = ` on every row: its left side already has a
  colon (`name: Type [metatype]`), so `: ` there would read
  `listen_port_: atomic<int> [atomic]: 8830`. Every row of that view, element
  rows (no declared type) included -- a first cut gave element rows `: `
  and typed rows ` = ` in the same view.
- The two-space version (previous section) is superseded, never committed.
- Tests: the types-view tests' expectations are back to ` = `; rows.mjs
  (compact) reads `name: value`. All 12 browser tests pass; screenshot
  reviewed.

## Subscription boxes get a fill: pale rose, 2026-10-03

Umbrella `cd8717c8`. `index.html` only. RC asked how box colours are
chosen -- a fixed palette per box KIND (`layout()`'s `kind`, a CSS class;
pale fill + darker outline of one hue), hand-picked in the page's first
version (`3c60fca4`); not derived from the C++ type, so a new kind would
fall to SVG's default black fill. RC: give WsSessionRouter::Subscription
(the "subscription" kind, the one white box) a pale colour.

- `.node.subscription > rect`: fill `#fceef3` (pale rose -- a hue no other
  kind uses, so not confused with stream green or session lilac); outline
  unchanged (`#7a4fa0`, its session's purple). Screenshot reviewed: an open
  subscription (60% fill opacity) is fainter still, but distinct.

## Unquoted values for non-string types, 2026-10-03

Umbrella `23a9817f`. The introspect page only (`introspect.js`).
RC: atomic values whose type is not a string should drop the quotes, e.g.
DynamicEndpoint::kind_.

- A scalar row whose json value is a string keeps json's quotes only when
  its declared type (`_canonical_type_`) is a string type --
  `is_string_type()`: the type ITSELF is `basic_string<..>` /
  `basic_string_view<..>` / `char*` (const either side) / xo `flatstring<..>`.
  Anchored at the start, so `deque<string>` is not (a first cut matched the
  `basic_string` inside it). Otherwise bare: `kind_ = stream`,
  `state_ = running`, `uri_regex_ = 0 captures`, `http_handler_ = set`,
  `outbound_q_ = 0 queued` -- enums, and printers' summaries.
- A row with no declared type (an element row) keeps json's rendering.
  Arrays of scalars (e.g. `var_v_ = ["name"]`) keep json's rendering too --
  a vector of strings, quoted rightly; a vector of enums would stay quoted
  (none today).
- Checked: the predicate on 10 spellings (gcc / clang char*, string_view,
  flatstring, deque<string>, enums); browser tests' expectations follow
  (`endpoint_expand.mjs`: `kind_ = stream` bare beside `uri_pattern_ =
  "/introspect"` quoted). All 12 pass.

## The box you click in stays put, 2026-10-03

Umbrella `4cd651c3`. The introspect page only (`introspect.js`).
RC: clicking in a box (e.g. a Subscription's endpoint_ ▾) re-lays out the
graph and the box "runs away from the mouse"; anchor it -- or move the
mouse along. Browsers cannot move the pointer (no API; Pointer Lock only
hides it), so the drawing moves instead.

- ANCHOR: a click or keydown inside a box (capture listeners on the svg:
  the box, its rows, triangles, the children toggle beside it) or one of its
  menu items (`show_menu()` -- the menu is outside the svg) names that box;
  the next `draw()` consumes it.
- After ELK, `anchor_shift()`: per axis, need = where the box was drawn -
  where ELK put it; `shift` (the drawing's offset in the svg, >= 0: nothing
  left of / above the svg) = max(0, need); the page scrolls by shift - need.
  On screen a box sits at svg_left + x - scrollX, so it stays put. When
  that scroll is past the page's end, the svg grows (`svg_min`) to make the
  room -- otherwise the browser clamps the scroll and the box moves (seen:
  16 and 85 px, in the first cut).
- `shift` and `svg_min` persist across unanchored redraws (Refresh), so
  they don't jump; Show all / Hide all reset both.
- Checked in headless chrome (`anchor.mjs`, new, real mouse clicks):
  opening a sub box from its label; its endpoint_ ▾ (hides /demo/) and ▸
  (shows it); opening the server; session 2's children triangle; session
  1's menu "Hide children" -- each time the box clicked in is at the same
  screen position (< 1 px) while the layout changes; Refresh moves nothing;
  Show all resets the shift. All 13 browser tests pass.

## Step 3, 2026-10-03 -- transitions

Umbrella `f5a15096`. The introspect page only (`introspect.js`,
`index.html`). RC: start on transitions; edges option (a), fade.

- Every redraw animates, three phases: leaving boxes and all edges fade out
  (`t_fade` 120 ms, while ELK lays out); boxes slide to their new places
  (`t_move` 350 ms, cubic in-out) and outlines / badges resize; then arriving
  boxes, rows that appeared, and the edges' new routes fade in (`t_show` 150
  ms). The svg grows at once, shrinks only at the end (nothing clipped
  mid-move). A new draw interrupts the last one's transitions and carries
  on from where things are.
- Edges fade rather than morph (RC: (a)): old and new orthogonal routes have
  different bends; morphing polylines makes spaghetti.
- Leaving boxes are inert while fading (`.node.leaving`, no pointer
  events); one that comes back mid-fade has its fade stopped. Rows that go
  disappear at once (their box shrinks over t_move) -- not faded.
- An arriving box is invisible until placed: it is joined before the async
  ELK layout, so would otherwise flash at (0,0). Arrival is marked by an
  element PROPERTY (`__arriving`), not a class: the first cut used a class,
  which `node.attr("class", ..)` rewrote before placement -- no box was ever
  seen as arriving, all stayed at opacity 0.
- With anchoring (above), the anchored box doesn't move; the rest glide
  around it.
- `settled()`: a promise resolved once the latest draw's transitions end
  (+50 ms slack: d3 starts them on its next tick), for tests.
- Checked in headless chrome (`transitions.mjs`, new; 3 runs green):
  session 2 slides through 11 positions (y 336 -> 452) when the server
  opens; edges transparent mid-move, opaque once settled; a hidden box fades
  (0.48..0.74 mid-fade) then is removed; an arriving box starts at 0 and
  ends at 1; open-then-close within 80 ms ends closed, at its final place,
  fully shown; no box left half-faded. The 13 other browser tests pass
  unchanged.

## Refcount badges dropped, 2026-10-03

Umbrella `9709b13d`. The introspect example: `introspect.js`,
`index.html`, `introspect.cpp`. RC: drop the refcount badge --
insufficiently interesting detail.

- Page: no badge (drawing, CSS); no refcount accounting in `layout()` (the
  "expected" holds a badge was compared with, turning it red); `badge_r`
  -- the room left above boxes for a badge -- gone from placement, svg size,
  edge paths and `anchor_shift()`.
- C++: `IntrospectSnapshot::app_holds_` (its comment: "for refcount
  accounting only") removed, with `Ticker::visit_sinks()` (its only use) and
  `IntrospectReceiver`'s Ticker pointer; and the `JsonMembers.hpp` include,
  there only for `json_id`.
- The library printers' `refcount` fields stay in the json ("Show JSON").
- Closes the open "red /introspect badge (4 vs 3)" item: no badge to be red.
  Its cause (likely the endpoint held during `receive`) was never confirmed.
- Checked: clean build, no warnings; ctest 49/49; `noticker.mjs` now checks
  no badges, no `app_holds` in the snapshot, `server.refcount` still in the
  json. All 14 browser tests pass.

## The graph gets its own viewport and camera, 2026-10-03

Umbrella `e88d39da`. The introspect page only (`introspect.js`,
`index.html`). RC: after anchoring, parts of the page -- the controls at
the top, the "last frame" json -- end up left of the viewport; "the graph
should be drawn in its own canvas, so that it can have a coord transform
applied to it separately from outside-the-graph display elements".

- Cause -- a modelling error in "The box you click in stays put" above:
  when the anchor had to move left / up, it scrolled the WHOLE PAGE
  (`window.scrollTo`) and grew the svg so the page could scroll that far.
  The page scroll stood in for a camera the graph didn't have; everything
  else on the page moved with it.
- Now: the svg is a fixed viewport -- full width, height filling the browser
  window below the controls (RC), resized with the window. One group,
  `g.camera`, holds edges and boxes and carries a d3.zoom transform: drag
  the background to pan (a drag starting on a box is the box's), the wheel
  to zoom (0.2..3), a new "Fit" button for the whole drawing (scale <= 1)
  (RC: option (a), pan + zoom).
- Anchoring moves the camera: `camera_target()` -- translate by (old - new
  position) * k -- as a transition in step with the boxes' own move, same
  duration and easing, so the anchor stays put on screen throughout. No
  anchor: the camera stays (Refresh); Show all / Hide all / first draw reset
  it to the origin. Gone: `shift`, `svg_min`, the page scroll, and the
  clamping (negative coordinates are fine behind a camera).
- Checked in headless chrome: `anchor.mjs` -- all its anchoring checks hold
  through the camera; the page never scrolls and the controls never move;
  a background drag of (-100, -50) pans the graph by exactly that, not the
  page; a drag from a box does not pan; the wheel zooms (k 1 -> 1.5); Fit
  puts the drawing inside the viewport; Show all resets the camera; the
  viewport's bottom is within 30 px of the window's. All 14 browser tests
  pass.

## Zoom readout beside the controls, 2026-10-03

Umbrella `e88d39da`. RC first saw "a
very long viewport, the Webserver box invisible" -- then: the mouse wheel,
used to scroll the page, now zooms the graph (wanted, but new), shrinking the
drawing out of sight. Asked for a magnification readout outside the
viewport, among the controls.

- `#zoom-level` after Fit: "zoom 100%", updated on every pan / zoom (the
  d3.zoom handler); its tooltip says the wheel zooms the graph and dragging
  its background pans.
- Ruled out on the way: a stale cached page (the server sends
  `cache-control: no-store`). A missing Fit button made the script throw
  at load (simulated in headless chrome) -- now null-guarded.
- Checked in headless chrome: `anchor.mjs` -- after a wheel zoom the
  readout reads `zoom 152%` (k 1.516); after Show all, `zoom 100%`.
  `cdp_menu.mjs` passes. Controls-bar screenshot reviewed.

## Panning / zooming keeps a box in view, 2026-10-03

Umbrella `e88d39da`. RC: prevent
panning that takes the entire drawing out of view -- the centre of at least
one box must stay inside the viewport.

- `keep_a_box_in_view()`, d3.zoom's `constrain`: every gesture's proposed
  transform passes through it. If some box's centre lands inside the
  viewport, accepted as is; else the transform is corrected just enough to
  put the centre NEAREST the viewport on its edge -- the drawing sticks
  there (half that box showing), and dragging back moves at once. The
  wheel is held to it too (zooming in on empty space, or far out toward a
  corner). `drawn_at` now carries each box's size, for its centre.
- Not constrained: the page's own camera moves (anchoring, Fit, the reset
  on Show all / Hide all) -- d3's `zoom.transform` bypasses constrain; they
  keep the drawing in view anyway.
- Checked in headless chrome (`anchor.mjs`): four hard drags up-left leave
  one box centre in view, at the viewport's edge; dragging back (+40, +30)
  moves the graph by exactly that; 8 hard wheel-ins on an empty corner
  (zoom 300%) still leave a box centre in view. All 14 browser tests pass.

## Pan / zoom only with Shift, 2026-10-03

Umbrella `e88d39da`. RC: pan and
zoom only with Shift pressed; otherwise let events fall through to the
browser.

- d3.zoom's filter now requires `shiftKey`: without it d3 takes nothing --
  the wheel scrolls the page, a drag is the browser's. Shift + drag on the
  background pans (a drag from a box still doesn't); Shift + wheel zooms.
- Browsers treat Shift+wheel as a sideways scroll; some report it in
  `deltaX` with `deltaY` 0, which d3's default wheel delta ignores. Custom
  `wheelDelta`: d3's formula on `deltaY || deltaX`.
- The hand cursor shows only while Shift is held (`svg#graph.panning`,
  toggled on keydown / keyup, cleared on window blur). The zoom readout's
  tooltip names the gestures.
- Checked in headless chrome (`anchor.mjs`): a drag WITHOUT Shift does not
  pan; the wheel without Shift scrolls the page (scrollY 200), no zoom;
  Shift + drag pans by exactly the drag; Shift + drag from a box doesn't;
  Shift + wheel zooms -- also when reported as deltaX (k 1.52 -> 2.30);
  the constraint checks pass with Shift. All 14 browser tests pass.

## Rows slide with their box's outline, 2026-10-03

Umbrella `937a9a52`. The introspect page only (`introspect.js`).
RC: opening a whole box is right (old text fades, new text appears once the
outline is ready), but opening a NESTED element draws text outside the box
until the transition catches up.

- Cause: opening a box, every row is new -- invisible until the outline has
  grown. Opening a nested row, the rows BELOW it already exist; the sizing
  pass set their new y at once, while the outline grew over t_move: for that
  time they sat below the box's old bottom. Same sideways: a nested row can
  widen the right-justified name column, moving existing rows right at once.
- Fix: before a draw re-renders a box's rows, each row already on screen
  records where it was (`__was`: text x, y, separator column -- the
  separator tspan is rebuilt each draw, so it must be read first). After the
  sizing pass, `slide_rows()` puts every moved row back there and transitions
  it to its new place -- t_move, cubic in-out, the outline's own duration
  and easing (now explicit there too), so a row inside the old outline and
  inside the new stays inside throughout. The triangle squares (each now
  knows its row, `__text`) move by the row's y and separator-column change.
- Checked in headless chrome (`transitions.mjs`, sampling every 20 ms that
  no visible row's bottom / right passes its box's): opening url_router_
  (nested), opening session_table_ below it, closing url_router_ -- worst
  0 px, 45 samples each; session_table_'s row slides through 12 positions.
  The same checks with `slide_rows` switched off: 27 px outside, and a
  1-position jump -- so they do catch it. All 14 browser tests pass.

## Legend: one colour per C++ type, 2026-10-03

Umbrella `e870e482`. The introspect page only (`introspect.js`,
`index.html`). RC first proposed showing each box's short type inside it
(a small grey line above the label, the menu button beside it); then,
before any code: a legend pairing each colour with its short type instead.
No in-box type line was built (my recommendation: the legend makes it
redundant; revisit if colours prove not enough).

- Colour is per box KIND, and kinds and C++ types were not one-to-one: http
  and stream endpoints are both DynamicEndpoint, in amber and green. RC: drop
  that distinction -- every endpoint is now amber (the dashed green "uses"
  edge to a stream endpoint keeps its green).
- `#legend`, one line between the source-link line and the viewport (not
  in the camera: it doesn't pan or zoom): per colour a swatch and the short
  type -- `WebserverImpl`, `DynamicEndpoint`, `WebsocketSessionRecd`,
  `WsSessionSender<WebserverImpl>`, `Subscription` -- taken from the
  snapshot's `_short_type_` (not hard-coded), with the canonical type as
  tooltip. Every kind in the snapshot, drawn or not, so it doesn't flicker
  as boxes show and hide. `size_view()` re-runs after it fills, so the
  viewport still ends at the window's bottom.
- Swatches are styled by the boxes' own CSS rules, which now match
  `:is(.node, .swatch).<kind> > rect` (same specificity as before). A first
  cut gave the swatch's group class `node <kind>`: every `g.node` lookup
  (`document.querySelector("g.node.server")`, in most browser tests) then
  found the legend's swatch -- 11 tests failed. Class `swatch` instead.
- Checked in headless chrome (`noticker.mjs`): the five types in order;
  each swatch has its boxes' fill; http and stream endpoints one colour;
  the viewport still ends within the window. All 14 browser tests pass.
  (Twice now a test printed nothing in a batch run and passed alone, twice
  over -- transitions.mjs here, visibility.mjs before: the harness, not the
  page, not investigated.)

## "Center" button, 2026-10-03

Umbrella `e870e482`. RC: a button
centring the graph in the viewport.

- `#center`, after Fit: at the current zoom, the camera moves (animated,
  as Fit) so the drawing's centre -- its laid-out bounds' -- is at the
  viewport's centre. A page camera move, so not held to "keep a box in
  view": zoomed far in, the centre can be empty space, no box in sight.
- Checked in headless chrome (`anchor.mjs`): after a Shift+wheel zoom and a
  pan, Center puts the drawing's centre within 1 px of the viewport's, zoom
  unchanged (k 2.27). `cdp_menu.mjs` passes.

## First draw centred, 2026-10-03

Umbrella `e870e482`. RC: centre the
diagram automatically when it is first drawn.

- `camera_target()` (now returning `{t, instant}`): on the first draw, the
  drawing's laid-out bounds centred in the viewport at scale 1, applied at
  once (no transition: nothing on screen yet to move from). Show all / Hide
  all still reset to top-left -- not asked; easy to centre too.
- Checked in headless chrome (`anchor.mjs`): first draw, the drawing's
  centre within 1 px of the viewport's, k = 1; a fresh-load screenshot
  shows the Webserver box mid-viewport. All 14 browser tests pass.

## Sink boxes; cycle breaking by model order, 2026-10-04

Umbrella `a14dbcab` (together with RC's WebsocketSink
SelfTaggingDisplayable change -- see .xo-backlog/xo-webutil/issues/01).
RC: with WebsocketSink self-describing, draw sink boxes.

- `WebsocketSinkImpl::print_json` now writes `_members_` (JsonMembers):
  `sender_` (a ref -- the sender is printed in full under its session),
  `pjson_`, `stream_name_`, `sub_id_`, `n_in_ev_`. Its existing keys are
  unchanged.
- Page: each subscription with a sink owns a `sink` box (labelled "sink",
  small, pale slate, legend entry `WebsocketSinkImpl`), hidden by default;
  the subscription's `sink_` row is a drawn ref (▾ (→), an edge); the sink
  box opens to its rows, `sender_` drawing an edge to the session's sender.
  12px text for the small receiver and sink boxes, as sender / subscription.
- Layering broke: with Show all and a sender box open, session 1 laid out
  ABOVE the Webserver (y 43 vs 443) -- sink -> sender and sender -> server
  back-edges outweighed the ownership edges' priority. Tried in the tab
  (ELK options overridden, no file changed): ownership priority 1000 -- no
  effect; cycleBreaking DEPTH_FIRST -- server on top, but the sender below
  the sink; MODEL_ORDER -- server, sessions, sender just below its session,
  sink below its subscription. Chose `elk.layered.cycleBreaking.strategy:
  MODEL_ORDER`: ELK reverses edges pointing from a later box to an earlier
  one in input order, and `layout()` lists parents before children -- so
  ownership decides layering by construction.
- Tests: `sink.mjs` (new): the sink's members listed; hidden by default,
  shown by its subscription's triangle; sink_ ▾ (→) with an edge; the box
  opens, sender_ ▾ (→) with an edge to the sender, stream_name_ "/demo/1";
  pale slate, in the legend. Expectations moved: legend seven types; Show
  all 14 boxes; sub_expand's sink_ a drawn ref (two edge ends, compared as
  a set -- their order changed with the layout); router_expand picks the
  session's edge into the sender (the sink's is a second); cdp_menu presses
  Fit before real clicks (session 2 now below the viewport's bottom -- the
  click missed and the checks read a stale menu) and checks the button's
  drawn size (Fit zooms out). ctest 49/49; all 16 browser tests pass.

## WebserverConfig opens on the page, 2026-10-04

Umbrella `a14dbcab` (with the sink section above). RC: make
WebserverConfig "displayable" -- clarified: openable on the page (1(a)),
not ref::Displayable (which derives from Refcount: wrong for a value type).

- WebserverConfig, a value class fully reflected, was printed by the
  generic struct printer: its fields as plain json keys, no `_members_` --
  so the Webserver's `ws_config_` row read `WebserverConfig` and could not
  open. Considered (1b): the generic struct printer writing `_members_`
  for every reflected struct -- deferred: it changes every reflected
  struct's json everywhere (python, tests).
- `JsonPrinter_WebserverConfig` (xo-websock, websock_json.cpp; befriended
  by WebserverConfig): `_name_`, type keys, `_members_` port_, tls_flag_,
  host_check_flag_, use_retry_flag_, mount_origin_. No `id`: a value,
  printed inside its Webserver, no box of its own.
- Checked: utest.websock (server members test) -- ws_config_'s value has
  those five members, a default config's port 0, tls false, mount origin
  "./mount-origin", no id; ctest 49/49. Headless chrome (`rows.mjs`):
  ws_config_ opens to `port_: <port>`, `tls_flag_: false`, ...,
  `mount_origin_: "..."`. All 16 browser tests pass. `xo-build --sweep
  -j 8`: 73 attempted, 73 ok (build); 47 ok + 26 with no tests; `--sweep ok`.

## Experiment: nested boxes, 2026-10-04

Umbrella: not yet committed. The introspect page only (`introspect.js`,
`index.html`). RC: try giving nested content (struct-valued members) its
own box -- an edge from outside can then arrive at the nested object itself
-- with holder and nested boxes inside an outline-only ELK box, so they
stay close. RC: nested boxes shown like refs (2a); a group only when a box
shows at least one nested box (3). Behind a "nested boxes" checkbox, off by
default, so both layouts can be compared on the same data.

- Spike first (elkjs 0.12, in the page): a compound node's children come
  back relative to it; each edge carries `container` -- the frame of its
  coordinates (`root`, or the group for an edge inside one). So absolute =
  container offset + reported.
- Model: `layout()` adds a `nested` box per struct-valued member
  (`is_nested_struct`: members of its own; not a ref, array or ref map),
  recursively, all in the top box's group; id = the holder's row key for
  the member, so expanded / wanted keys carry over. A `nests` edge makes it
  the holder's child (`is_ownership()` now covers link / owns / nests: the
  triangle and menu treat nested boxes as children). `all_refs()` stops at
  a nested struct and emits an edge to its box -- refs inside belong to it.
  `member_rows()` draws such a member as a ref row, ▸ / ▾ (→).
- ELK: `elk.hierarchyHandling: INCLUDE_CHILDREN`; `elk_children()` wraps a
  box with drawn nested boxes as `grp:<box>` (padding 12); positions made
  absolute; group outlines drawn under everything (`draw_groups()`,
  moving / fading like boxes). Legend entry "nested struct".
- Found on the first try: grey ownership edges back (server -> /hello,
  /types ...) -- the endpoints' member edges now leave the nested UrlRouter
  box, so the server had "no ref" to its children and the fallback ownership
  edge stood in. Fixed: an owner reaches a child through its nested boxes
  too (`within()`), and showing a child (triangle, menu, `show_box`) wants
  that edge AND the chain down to the nested box it leaves (`child_edges()`).
- Not (yet): array elements of structs still open in place; nested boxes all
  share one colour (white / grey); anchoring, transitions and keep-in-view
  with nested boxes on are untested beyond `nested.mjs`.
- Checked in headless chrome (`nested.mjs`, new): off -- 14 boxes, no
  groups; on + Show all -- url_router_, session_table_, ws_config_ and both
  routers get boxes, every earlier box still drawn; each group outline
  encloses its holder and nested boxes; UrlRouter's box receives edges from
  the server and both session routers; no ownership fallback edges; the
  server's url_router_ row ▾ (→), its ▾ / ▸ hide / show the box; nested
  boxes are children; off again -- exactly the earlier boxes, no groups.
  The 16 earlier browser tests pass unchanged (checkbox off).

## Nested boxes permanent, 2026-10-04

Committed in 3d0813ff. RC: "Let's make the experiment permanent". The "nested boxes" checkbox and
its flag are gone; every draw lays out nested boxes.

- Latent in the experiment, found once always on: array elements that are
  structs were treated as nested too. `nests(m)` now requires a declared
  member (`_canonical_type_` present) whose value has members and is not a
  ref, array or ref map. Array elements still open in place.
- Ownership follows the boxes (RC chose re-parenting): `reparent_via_nested()`
  makes a child that its owner reaches only through a nested box's ref the
  child of that nested box. A child the owner refs directly stays (a
  session keeps its sender). Result:
  - server -> ws_config_, url_router_, session_table_ (▸3, was ▸6);
  - url_router_ -> the endpoints;
  - session_table_ -> the sessions;
  - a session's router_ -> its subscription.

  This fixed a bug the experiment had: the UrlRouter box's menu offered
  no children at all, and the server double-counted (▸9: 3 nested boxes plus
  6 reached through them). Showing a deep box still pulls in the whole path.
  ELK's layering still uses the server's own link/owns edges.
- Tests rewritten for the always-on layout (scratchpad .mjs):
  - router_expand: router_ is its own box; the sender has three incoming
    edges (session, router, sink), not merged.
  - visibility: the counts and menus above.
  - rows: on the UrlRouter and WebserverConfig boxes. Live rows now nest at
    most 2 deep (depth 0-1), so the separator-step check needs 2 depths,
    not 3.
  - expand: the injected `nested_` struct is a box.
  - cdp_menu: the "Hide children" count comes from the box's `n_children`.
  - nested: no off/on comparison.
- anchor: the wider layout put session 1's menu button past the viewport's
  right edge (x 1444 of 1437). The test's real click missed, landing outside
  the svg, and every later Shift-drag near the window bottom then scrolled
  the page by 35px. That broke "dragging back" and "viewport fills the
  window". The test now presses Fit first. After the fix the page no longer
  scrolls; I did not dig into why the missed click caused the scrolling.
- Checked in headless chrome: all 17 browser tests pass (rows and receiver
  with --src-tree, the rest without).

## Nested boxes get a colour per type; legend on the canvas, 2026-10-04

RC: nested children have their own boxes, so each needs a colour, as the
parents have. Also: move the legend onto the canvas and list it
vertically. RC chose an automatic palette and a legend fixed in the
top-left corner.

- Colour: `nested_colour(type)` gives each canonical type a {fill, stroke}
  from `nested_palette` (8 pale fills no other kind uses: green, peach,
  cyan, olive, periwinkle, mauve, sage, sand). A type gets its colour the
  first time it appears, and the page keeps it, so a refresh never repaints
  and a new nested type needs no CSS. The colour is an inline style on the
  box's rect. Boxes keep kind `nested`; the CSS `.nested` rule (white/grey)
  is only the fallback.
- Legend: `g.legend` in the svg, outside `g.camera`, so pan and zoom leave
  it alone. It sits over the drawing, 8px in from the viewport's top-left,
  on a white panel at 85% opacity. One entry per row: swatch, short type,
  and the canonical type as tooltip. The 7 kinds come first, then one entry
  per nested type in the order they first appear; the old "nested struct"
  entry is gone, and so is the `#legend` div above the graph.
- Camera: `legend_w` (the panel's width plus margins) is a column the
  drawing leaves free when the camera is placed. Fit, Center, the
  first-draw centring and the reset (Show all) all keep the drawing right
  of it. Panning can still slide boxes under the legend.
- Seen in a screenshot, not changed: some nested colours sit close to
  existing kinds. WsSessionTable's cyan is near IntrospectReceiver's teal,
  and WsSessionRouter's olive is near the sender's beige.
- Checked in headless chrome (noticker extended):
  - the legend lists the 7 types, then WebserverConfig, UrlRouter,
    WsSessionTable<…> and WsSessionRouter;
  - entries form one column;
  - the 4 nested colours are distinct and match no parent kind's colour;
  - every nested box matches its swatch, and both routers share one
    colour;
  - the panel stays at the corner, same size, under a pan + zoom;
  - after Fit the drawing starts right of the legend.

  anchor's first-draw, Center and reset checks now measure from the
  legend's column. All 17 browser tests pass.

## Edge decoration: direction, crossings, how a holder relates, 2026-10-04

Committed in e2b095cc (arrowheads, rounded bends, casing) and 9c7b5ec2
(exit markers, with the 25% shrink).

RC wanted to show direction and make crossings readable, and chose
options 1-3: arrowheads, rounded bends and casing. RC also asked for a mark
where an edge leaves its box, to tell inclusion, ownership and reference
apart. RC: keep "shares" separate, with an open circle for it.

- Arrowheads: `define_arrowheads()` adds svg markers, one per edge colour
  (member purple, hot orange, uses green, fallback grey). index.html sets
  them per class with `marker-end`. Their size is in drawing units, so the
  3px hot stroke doesn't grow them.
- Rounded bends: `rounded_path(pts, 6)` draws each bend as a quadratic
  curve, of radius at most 6 and less on short segments. The last segment
  stays straight, for the arrowhead.
- Casing: each edge is now a `g.edge-g` holding a `path.casing` (5px, the
  graph's background colour) and then the edge. Where a later edge crosses
  an earlier one, the earlier shows a gap. The fades run on the group,
  and `highlight_ref` raises the group. Inside a group's grey panel the
  casing shows as a faint lighter halo.
- Exit markers (`marker-start`): `ref_kind()` reads the declared canonical
  type; an element uses its container's type, and the outermost smart
  pointer wins:

  | type | kind | mark |
  |---|---|---|
  | nested struct (by value) | includes | ■ |
  | `std::unique_ptr` | owns | ◆ |
  | `intrusive_ptr` (rp), `std::shared_ptr` | shares | ○ |
  | anything else (T*, T&) | refers | none |

  Merged parallel edges take the strongest kind. Each edge's path gets a
  `from-<kind>` class, and its tooltip ends with "(kind)". The legend
  lists the four kinds under the colours, as sample lines (`path.sample`,
  deliberately not `.edge`: the edge lookups would otherwise find them).
- What the live snapshot gives, checked via `layout(last_event)`:
  - includes: ws_config_, url_router_, session_table_, router_;
  - owns: session_map_ elements, subscription_v_ elements;
  - shares: http_map_/stream_map_ elements, receiver_, every sender_,
    endpoint_, sink_;
  - refers: the routers' url_router_ (const UrlRouter&) and the sender's
    target_ (WebserverImpl*; never drawn, since it points into the
    Webserver).
- Tests: `edge_kinds.mjs` (new) checks:
  - each drawn edge's kind against the table above;
  - the computed marker-start per kind, and that refers has none;
  - the arrowhead;
  - that both markers turn orange on hover;
  - the legend block's order and placement;
  - the tooltip.

  transitions now reads the group's opacity, and visibility's tooltip
  check expects the kind suffix. All 18 browser tests pass.
- Follow-up, RC: the exit markers were a little large, so shrink them by
  25%. They are now drawn at `start_scale` = 0.75 of their 12x10 viewBox,
  9x7.5 drawing units. Their outline is scaled up to match, so it stays as
  wide as the edge (1.4). edge_kinds.mjs still passes.

## Edge colour by ref kind, 2026-10-04

RC: try colouring edges by ownership, from black to light grey, with
includes black and refers light grey.

- Member edges are no longer all purple. A grey ramp, darker the closer the
  holder holds its target, lives once as CSS variables on `:root` in
  index.html:
  - `--edge-includes` `#1a1a1a`;
  - `--edge-owns` `#555555`;
  - `--edge-shares` `#8c8c8c`;
  - `--edge-refers` `#c4c4c4`.

  Each `from-<kind>` class sets the stroke, `marker-end: arrow-<kind>` and
  (for all but refers) `marker-start: start-<kind>`.
- `define_arrowheads()` fills the per-kind arrowheads and exit markers with
  `var(--edge-<kind>)`, so a colour has one source. The `-hot` exit
  markers and `arrow-hot` stay orange, so hovering still works as before.
  The green "uses" edges and the grey fallback edges are unchanged.
- The legend's edge-kind samples take the same rules: colour, exit marker
  and arrowhead.
- Tests: edge_kinds.mjs now checks each kind's computed stroke, its
  arrowhead (id and fill), the filled exit markers' fill and the open
  circle's outline. All 18 browser tests pass.

## Shared entry port: the strongest kind on top, 2026-10-04

Problem: every edge into a box enters at its one port, so the edges overlap
on their last stretch. The edge drawn last won: its line, casing and
arrowhead covered the rest. That order was ELK's output order, so the
trunk's colour was arbitrary. RC asked whether the colour could be
chosen, and chose option 1, "strongest on top", over one entry port per
kind.

- `order_edges()` sorts the edge groups by `edge_rank`:
  - non-member edges (uses, fallback ownership) at the bottom;
  - then refers < shares < owns < includes;
  - the sort is stable, so equal ranks keep their order.

  It runs after each edge join, and in `highlight_ref` when the hover
  ends: hovering still raises the edge, and leaving restores rank order.
- Example: three edges enter the UrlRouter box. The server's includes edge
  (black) now lies over both routers' refers edges (light grey), so the
  trunk and arrowhead read black.
- Tests: edge_kinds.mjs checks that the DOM order is non-decreasing in rank,
  that the UrlRouter's three incoming edges stack as refers, refers,
  includes, and that hover raises an edge and leaving restores the order.
  All 18 browser tests pass.

## "uses" edges dropped, 2026-10-04

Committed in 5345640f.

RC: drop the "uses" feature, the green dashed subscription -> endpoint
edge. It repeated the subscription's endpoint_ member edge, which already
shows the same link (a shares edge, ○).

- `layout()` no longer adds `edge(sid, ep, "uses")`. Removed with it: the
  `endpoint_node` map (it existed only for that edge), the
  `path.edge.uses` CSS, and the `arrow-uses` marker. Without the uses
  edges ELK lays the graph out a little differently.
- cdp_menu.mjs runs in headless chrome's default 780x493 window, and after
  Show all the server box now landed past the viewport's right edge (x 1073
  of 741), so the right-click missed. The test now presses Fit after Show
  all. All 18 browser tests pass.

## Box header: the short type, then the box's title, 2026-10-04

RC: label boxes consistently. Every box's first line names its short type;
a box with a title of its own (/hello/${name}, sub 0 · /introspect) puts
that on the second line. RC took the proposal for which boxes get a
second line, and asked for it smaller but not grey.

- Line 1 (`text.label`) is `box_type_label(d)`, the object's
  `_short_type_`. Line 2 (`text.sub`) is `box_subtitle(d)`:
  - server: ":<port> (<state>)" ("Webserver" would repeat WebserverImpl);
  - endpoint: its pattern; session: "session N"; subscription:
    "sub N · <stream>";
  - sender, sink, receiver, nested struct: none. "sender" and "sink" only
    restated the type; a receiver's and a nested box's label *is* their
    type.
- Line 2 is one size down (12px; 11px in small boxes), in the type's
  colour. The menu button and the children triangle stay level with
  line 1. A box with a title is taller by `sub_h` (16; 14 when small), and
  its rows start below the title. Width is the wider of the two lines.
  `d.label` is unchanged, so menus and tooltips still name a box by its
  title ("Hide session 1", "Show ▸ /introspect").
- The taller boxes widened the drawing. Three tests clicked past the
  viewport's right edge with the real mouse:
  - rows: the UrlRouter's triangle, at x 1461 of 1461. The test now pans
    at 100% first, since it measures pixels.
  - anchor: session 2's triangle, at x 1673. That check had been passing
    only because the click missed, and the missed press then made the
    later Shift-drags scroll the page. Fit now runs before that step, plus
    a check that session 2 lies inside the viewport.
  - session_expand, router_expand: they found session 1 by its title,
    which now sits in `text.sub`.
  - sink: it expected the title "sink"; the box now shows only its type.
- header.mjs (new) checks:
  - every box's line 1 is its short type;
  - each kind's line 2, and the sender, sink, receiver and nested boxes
    having none;
  - font sizes 14/12 and 12/11, with line 2 in line 1's colour;
  - the menu button and triangle level with line 1, line 2 below it;
  - an open box's rows below line 2;
  - the "Hide session 1" menu item.

  All 19 browser tests pass.

## Drawing extent shown; Center ignores the legend; legend checkbox, 2026-10-04

RC asked how Center is defined, since it didn't behave as expected.
Answered from the code: Center keeps the zoom and puts the centre of
`drawing_size` (ELK's laid-out size plus `pad`) at the centre of the
viewport area right of the legend. RC then asked:
1. to show the ELK drawing's extent, set apart from the viewport the way
   the legend's panel is;
2. for Center to use the whole viewport's centre, ignoring the legend;
3. for a checkbox that controls whether the legend is drawn.

- Extent: `draw_extent()` puts a rect of `drawing_size` as the first child
  of `g.camera`, under the groups, edges and boxes. It is white with a 1px
  `#dcdcdc` border (`vector-effect: non-scaling-stroke`) on the viewport's
  `#fafafa`, and resizes with the boxes' `size` transition. The edge
  casing and the open-circle exit marker's fill changed from `#fafafa` to
  white, to match the sheet behind them.
- Found via the sheet: Fit, Center and the first-draw centring took the
  viewport's size from `getBoundingClientRect()`, which includes the
  svg's 1px border. Fit therefore scaled the drawing about 2px past the
  inside edge; the sheet's border made that visible. `view_size()` now
  uses `clientWidth`/`clientHeight`.
- Found while testing: a bug that had been showing up as "the page scrolls
  35px during a drag" (the anchor test's missed-click knock-on, twice
  before). If any text in the graph is selected, Shift + press counts to
  the browser as extending the selection. d3.zoom stops a new selection
  starting, but not an existing one being extended, so a Shift-drag pan
  toward the window's bottom scrolled the page. Fix: a capture-phase
  mousedown listener on the document. On Shift + press over the graph,
  outside a box, it calls preventDefault and clears the selection. It
  has to be capture: d3.zoom's own handler stops propagation on the svg.
  anchor.mjs now selects graph text first and checks that the Shift-drag
  neither scrolls the page nor leaves a selection.
- Center: `translate(width/2 - k*w/2, height/2 - k*h/2)`, with no legend
  term.
- Legend checkbox: "legend" (`#show-legend`), on by default, beside
  "types". Unchecked, the legend gets `display="none"` and `legend_w` = 0,
  so Fit, the first draw and the reset use the whole viewport.
- Tests:
  - noticker checks that the sheet is `drawing_size`, sits first in the
    camera, is white on the viewport's grey, matches the casing's colour,
    and pans and zooms with the camera;
  - noticker also checks the checkbox: on by default; off hides the
    legend, gives `legend_w` 0, and puts Fit at x 0; on again brings the
    legend back;
  - anchor's Center check now measures against the whole viewport.

  All 19 browser tests pass.

## Show all, Hide all, first draw: centred, 2026-10-04

RC asked how the starting location is picked. Answered from
`camera_target()`:
- first draw: centred right of the legend, at 100%, at once;
- Show all / Hide all (`camera_reset`): top-left, just right of the
  legend, at 100%;
- a click: the anchor box stays put;
- otherwise: the camera stays.

RC: "For now let's have show all, hide all, first draw all center."

- `centred(size, k)` gives the camera that centres a drawing of `size` at
  zoom `k` in the viewport (`view_size()`), with the legend not counted.
  Center uses it at the current zoom. The first draw uses it at k = 1,
  instantly; a reset uses it at k = 1, animated. Only Fit still reserves
  the legend's column (`legend_w`).
- A drawing wider than the viewport, centred at 100%, has both sides off
  screen. Two more tests made real clicks there after Show all
  (session_expand opening session 1, expand opening the server) and now
  press Fit first. expand's "box grew" check compared on-screen heights,
  so it now divides by the zoom.
- anchor's first-draw check now measures from the viewport's inner centre.
  Show all is checked as centred at 100%, and so is Hide all, starting from
  a pan to (-200, 90) at 60%. All 19 browser tests pass.
