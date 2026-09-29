# 13 — introspect: a context menu on each box

Status: open -- increment 1 built (uncommitted); refs / expand to come
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

## Increment 1, 2026-09-29

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
