# 11 — xo-websock gets an Appcx; json printers registered there

Status: done 2026-09-27, umbrella `f9dbfb08`
Type: refactor
Raised: 2026-09-27 (RC), from `.xo-backlog/xo-websock/issues/10` (5a)

Adopt the application-context ("appcx") pattern for xo-websock, as the
subsystems below it have, and register xo-websock's json printers from its
appcx rather than from `Webserver::make`.

## Today (measured 2026-09-27)

- `provide_websock_json_printers(PrintJson *)`
  (`xo-websock/include/xo/websock/websock_json.hpp`, issue 10 increment 5a)
  is called by `Webserver::make` on the PrintJson it is handed
  (`xo-websock/src/websock/Webserver.cpp`, `Webserver::make`). A side effect
  of making a server -- and each `make` re-installs (harmless: PrintJson keeps
  the first printer per type, `PrintJson::provide_printer`).
- xo-websock has no subsystem tag, no `init_websock.hpp`, no `cx/`:

  ```bash
  ls xo-websock/include/xo/websock/ | grep -i "init\|cx"      # nothing
  grep -rn "S_websock_tag" --include=*.hpp .                  # nothing
  ```
- Subsystems that have the pattern:

  ```bash
  ls -d xo-*/include/xo/*/cx
  ```
  -> xo-facet, xo-indentlog2, xo-interpreter2, xo-object2, xo-printjson,
  xo-reflect, xo-stringtable2.
- Another subsystem registers printers differently: xo-kalmanfilter from its
  `InitSubsys` (`xo-kalmanfilter/src/kalmanfilter/init_filter.cpp` ->
  `EigenUtil::provide_json_printers`).

## The pattern (from xo-printjson, the closest model)

`xo-printjson/include/xo/printjson/cx/PrintJsonConfig.hpp`,
`PrintJsonAppcx.hpp`, `src/printjson/PrintJsonAppcx.cpp`:

- `init_<sub>.hpp`: `enum S_<sub>_tag {}` + `InitSubsys<S_<sub>_tag>`
- `cx/<Sub>Config.hpp`: config class + `SubsystemConfig<S_<sub>_tag>`
- `cx/<Sub>Appcx.hpp`: context class, holding `InitEvidence`, the config, and
  what the subsystem owns; a template ctor `(Deps & deps, const Config &)`
  pulling the contexts it depends on via `deps.template cx<S_dep_tag>()`;
  `visit_pools`; `SubsystemContext<S_<sub>_tag>`
- `xo-subsys/include/xo/subsys/AppContext.hpp`: `AppConfig<Tags...>` /
  `AppContext<Tags...>` build contexts in dependency order.

`Indentlog2Appcx` (the example RC pointed at) is itself marked "_not_ a model
to use as a general-purpose pattern" (thread-local state); `PrintJsonAppcx` is
the plainer shape.

## Proposal

- `xo-websock/include/xo/websock/init_websock.hpp`: `S_websock_tag`,
  `InitSubsys<S_websock_tag>`.
- `cx/WebsockConfig.hpp`: empty config to start.
- `cx/WebsockAppcx.hpp` / `src/websock/WebsockAppcx.cpp`: depends on
  printjson; its ctor calls
  `provide_websock_json_printers(deps.cx<S_printjson_tag>().print_json())`.
- `Webserver::make` stops installing printers.

## Open

- **`Webserver::make`'s signature.** It takes `rp<PrintJson>` today. Keep it
  (the appcx only registers printers), or take the appcx / pull PrintJson
  from it?
- **Who builds the appcx.** Callers of `Webserver::make`:

  ```bash
  grep -rn "Webserver::make\|make_webserver" --include=*.cpp . | grep -v "/.build/"
  ```
  -> `xo-websock/example/introspect/introspect.cpp`,
  `xo-websock/utest/Webserver.test.cpp`, `WebserverLive.test.cpp`,
  `websock_utest_main.cpp` (disabled kalman demo), and xo-pywebsock
  (`Webserver.make`, `make_webserver`). The utest main
  (`xo-websock/utest/websock_unit_main.cpp`) builds only an indentlog2 context
  today. Python: xo-pyprintjson enforces one `PrintJsonAppcx` per python
  instance (`xo-pyprintjson/src/pyprintjson/pyprintjson.cpp`); xo-pywebsock
  would need the same kind of arrangement.
- **Fail loudly if absent?** With registration in the appcx, printing a
  `Webserver*` without one yields invalid json (observed in issue 10's
  falsification of 5a). Whether `Webserver::make` should require evidence
  (`InitEvidence` / creation evidence) that the appcx exists.

## Decided (RC, 2026-09-27)

`Webserver::make(WebsockAppcx const & cx, WebserverConfig const &)` -- the
server takes its PrintJson from the context, so no server exists without the
printers (compile-time; precedent: `AllocFlywheel` takes `FacetAppcx`,
`xo-facet/src/facet/AllocFlywheel.cpp`). Replaces `make(config,
rp<PrintJson>)`; `make` no longer installs printers. Python mirrors
xo.printjson: `configure()` once per process.

## Progress

**2026-09-27.** Umbrella `f9dbfb08`.

- xo-websock: `init_websock.hpp`/`.cpp` (`S_websock_tag`; `require()` depends
  on printjson; `init()` empty), `cx/WebsockConfig.hpp` (empty),
  `cx/WebsockAppcx.hpp` + `src/websock/WebsockAppcx.cpp` (depends on
  `PrintJsonAppcx`; ctor calls `provide_websock_json_printers`; exposes
  `print_json()`, creation evidence, placeholder `visit_pools`).
- `Webserver::make(WebsockAppcx const &, WebserverConfig const &)`.
- Tests: new `xo-websock/utest/WebsockUtestAppcx.{hpp,cpp}`, the
  `FacetUtestAppcx` shape (RC, xo-facet/utest): owns the test binary's
  `AppContext<indentlog2, reflect, printjson, websock>`; `configure(cfg)` from
  main, `appcx()` for tests (throws, not asserts, if unconfigured -- NDEBUG
  in Release). `websock_unit_main.cpp` builds only a `UtestAppConfig`; tests
  make servers from `WebsockUtestAppcx::appcx().cx<S_websock_tag>()`. Shared
  by utest.websock and utest.websock.live. (Replaced a first cut,
  `websock_test_appcx.hpp` / `xo::ut::websock_appcx()`, never committed.)
  Example builds the same stack.
- xo-pywebsock: `WebsockConfig`, `WebsockAppcx`, `configure(config,
  printjson_appcx)` (once per process, throws on a second call);
  `Webserver.make(cx, ws_config)` / `make_webserver(cx, ws_config)` with
  `keep_alive<0,1>`. CMake: header dependency on xo_pyprintjson;
  `pkgs/xo-pywebsock.nix` gains xo-pyprintjson.
- **Latent xo-printjson bug, fixed:** installed `cx/PrintJsonAppcx.hpp` had
  `#include "PrintJson.hpp"` -- resolves only when the include path reaches
  `xo/printjson/`. xo-pywebsock was the first installed consumer without it:
  sweep stage 1 failed there (`fatal error: PrintJson.hpp: No such file`).
  Now `"xo/printjson/PrintJson.hpp"`, as the next line already spelled its
  sibling. No other `cx/` header has the pattern.

Verified: utest.websock 44/555, utest.websock.live 6/90 (3/3), umbrella
49/49, `xo-build --sweep --with-examples` ok (after the header fix); example
serves its snapshot via the context; python by hand: chain
indentlog2 -> reflect -> printjson -> websock configure, both makes,
second configure refused, old one-arg make -> TypeError;
`nix-build ci-nxfs.nix -A xo-pywebsock` ok (25 derivations).

**Noticed, not fixed:** the xo-websock nix package runs NO tests -- its
checkPhase prints `No tests were found!!!` (`pkgs/xo-websock.nix` passes no
`-DENABLE_TESTING=1`). The known doCheck-sweep gap; nix never runs
utest.websock / utest.websock.live.

## Done when

- xo-websock has `init_websock.hpp`, `cx/WebsockConfig.hpp`,
  `cx/WebsockAppcx.hpp`, following the xo-printjson shape
- json printers are registered by `WebsockAppcx`, not by `Webserver::make`
- every caller listed above builds the context; the introspect example,
  utest.websock, utest.websock.live and xo-pywebsock work
- `xo-build --sweep --with-examples` ok in both stages
