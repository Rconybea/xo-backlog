# 13 — introspect: a context menu on each box

Status: open (mechanism understood; menu contents and left-click undecided)
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
  elsewhere, Escape, or scroll. No library.

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
- Escape, a click elsewhere, or scrolling closes it
- left-click behaves as decided
