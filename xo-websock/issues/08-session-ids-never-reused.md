# 08 — websocket session ids are never reused

Status: done 2026-09-26, umbrella `ae263aa9`
Type: refactor / bugfix
Raised: 2026-09-26, from `.xo-backlog/xo-websock/issues/05` (step C)

A session id is today an index into `WebserverImpl::session_v_`
(`xo-websock/src/websock/Webserver.cpp`), and a closed session's id goes on
`free_session_id_v_` for the next connection to reuse. Anything still holding
the old id -- a sink the application kept past its session -- then addresses a
DIFFERENT client. Issue 05 step B guards against that with
`WsSessionSender::close()`; this ticket removes the hazard at its source.

## Decided (RC, 2026-09-26)

- **Ids come from a counter and are used at most once**, never reused under any
  circumstances.
- **`uint64_t`**, so the guarantee is stated rather than relying on 2^32
  sessions being unrealistic.
- **Sessions are held in `std::unordered_map<uint64_t,
  std::unique_ptr<WebsocketSessionRecd>>`**, replacing `session_v_` and
  `free_session_id_v_`.

## Measured, 2026-09-26 (reading `xo-websock/src/websock/Webserver.cpp`)

```bash
grep -n "session_v_\|free_session_id_v_\|next_session_id_\|uint32_t session_id" \
     xo-websock/src/websock/Webserver.cpp
```

- **Ids are assigned in two places that disagree.**
  `LWS_CALLBACK_HTTP_BIND_PROTOCOL` does
  `new OutputBuffer(vhd->next_session_id_++)`; then `notify_ws_session_open`
  OVERWRITES `vhd->next_session_id_` from the free list (or
  `session_v_.size()`). Only the counter should remain.
- **A session's record outlives the session.** `notify_ws_session_close` runs
  `unsubscribe_all()` and frees the id, but leaves the
  `WebsocketSessionRecd` in `session_v_` until the slot is reused. (This is
  why 05 step B could not give sessions an `rp<Webserver>` sender: the record
  would keep a cycle alive.) With a map, close erases the entry.
- **Data race, inferred from the code, not reproduced.**
  `WebserverImpl::send_text` is reached from application threads (a sink's
  sender, e.g. a reactor source delivering) and reads `session_v_` with no
  lock, while the service thread may `resize()` it in
  `notify_ws_session_open`. A reallocation under a concurrent reader is UB.
- **`send_text` asserts** (`assert(false)`) on an id beyond `session_v_`. With
  never-reused ids, sending to a closed session is ordinary; it must drop, not
  assert.
- `Webserver::send_text(uint32_t, ...)` is public
  (`xo-websock/include/xo/websock/Webserver.hpp`), but its only callers are in
  `Webserver.cpp` -- no binding in xo-pywebsock, no other subsystem:

  ```bash
  grep -rn "send_text" --include=*.cpp --include=*.hpp --include=*.py . | grep -v "^./.build"
  ```

## Proposal

- **One mutex over the session map.**
  - `send_text` (any thread): look up and call the record's `send_text` under
    the lock. The record's `send_text` only queues and wakes the service
    thread, so the lock is held briefly.
  - `notify_ws_session_close` (service thread): MOVE the record out of the map
    under the lock, then with the lock released `close_sender()`,
    `unsubscribe_all()`, and destroy it. Released first because an endpoint's
    unsubscribe may send, which would re-take the lock.
  - `perform_ws_cmd` (service thread): look up under the lock, run the command
    with it released -- handlers send. Safe because only the service thread
    erases.
  - `lws_write_pending_traffic` and `~WebserverImpl`'s close-every-sender
    backstop iterate under the lock.
- **Unknown id in `send_text` drops the text** (logged), no assert.
- **`uint64_t` everywhere an id travels:** `OutputBuffer`, `WsSessionSender`,
  `perform_ws_cmd`, `send_text` (incl. `Webserver.hpp`),
  `per_vhost_data__minimal::next_session_id_`. The `lwsl_user` partial-write
  message prints the id with `%u`; needs a 64-bit format.

## Effect on issue 05

Step C's "a retained sink cannot write into a later session reusing its id"
becomes structural. `close()` still matters -- the sender holds a raw
`WebserverImpl *`, so a sink retained past the SERVER must not reach it -- so
step C shrinks to "a closed sender drops".

## Testing

All of this is inside `WebserverImpl`, which today runs only with a live
socket; `utest.websock` never constructs a `Webserver`. Open: how to test the
map and the never-reused guarantee -- a real `Webserver` on a local port, or
pulling the id/session bookkeeping into a class testable without libwebsockets.
Same question as 05 step C.

## Files

- `xo-websock/src/websock/Webserver.cpp` -- `OutputBuffer`,
  `per_vhost_data__minimal`, `WsSessionSender`, `WebserverImpl` (members,
  `notify_ws_session_open` / `_close`, `perform_ws_cmd`, `send_text`,
  `lws_write_pending_traffic`, dtor), `LWS_CALLBACK_HTTP_BIND_PROTOCOL`
- `xo-websock/include/xo/websock/Webserver.hpp` -- `send_text` signature

## Progress

**2026-09-26 -- implemented** (RC: `uint64_t`; option 1, `WsSessionTable`).
Umbrella `ae263aa9`.

- New `xo-websock/include/xo/websock/WsSessionTable.hpp`: header-only template
  over the per-session record. `next_id()` (counter from 1, only increases),
  `insert(id, unique_ptr)`, `take(id)` (removes, hands the record over),
  `with_session(id, fn)` / `for_each(fn)` (fn runs under the table's mutex),
  `find_owner_thread(id)` (pointer returned with the mutex released; valid
  only where nothing can `take()` concurrently -- the service thread),
  `size()`. One mutex.
- `WebserverImpl` holds `WsSessionTable<WebsocketSessionRecd> session_table_`
  in place of `session_v_` / `free_session_id_v_`.
  - the id's ONLY assignment: `LWS_CALLBACK_HTTP_BIND_PROTOCOL`,
    `websrv->next_session_id()`. `per_vhost_data__minimal::next_session_id_`
    and the free-list overwrite in `notify_ws_session_open` are gone.
  - close: `take()` first, then with the lock released `close_sender()`,
    `unsubscribe_all()`; the record is destroyed at the end of
    `notify_ws_session_close` (before lws deletes the `OutputBuffer`).
  - `send_text` (any thread): `with_session`; a closed/unknown id is logged and
    dropped -- the `assert(false)` is gone. Fixes the unlocked `session_v_`
    read racing `resize()`.
  - `perform_ws_cmd`: `find_owner_thread`, command run unlocked.
  - `lws_write_pending_traffic`, dtor backstop: `for_each`.
  - ids `uint64_t` in `OutputBuffer`, `WsSessionSender`, `perform_ws_cmd`,
    `send_text` incl. `Webserver.hpp`; partial-write log prints `%llu`.
- New `xo-websock/utest/WsSessionTable.test.cpp`, 4 cases with a fake record:
  never reused over 50 open/close rounds in both close orders; a closed
  session unreachable by every route, including after a newer session opens;
  `with_session` reaches exactly its session; `for_each` live only.
  Falsified with a compiling recycling `next_id()` (`size() + 1`): the
  never-reused case fails at its first duplicate.
- NOT tested: `WebserverImpl`'s use of the table (live socket only).

utest.websock 30 cases / 439 assertions; umbrella 48/48; `xo-build --sweep`
ok in both stages.

## Done when

- session ids are `uint64_t`, assigned from one counter, never reused
- sessions live in an `unordered_map`; a closed session's record is erased
- all access to the map is locked; `send_text` to an unknown id drops
- `free_session_id_v_` and the second id assignment in
  `notify_ws_session_open` are gone
- a test pins the never-reused guarantee (approach open, see Testing)
- `xo-build --sweep` ok in both stages
