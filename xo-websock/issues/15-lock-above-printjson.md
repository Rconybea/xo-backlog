# 15 -- take locks above printjson, not inside printers

Status: open
Type: task
Milestone: reflection-driven-json

RC (2026-10-05): printjson does not lock; threading is handled by locking
above the scope of a printjson call. Today these printers take their own
mutex while reading:
- WsSessionRouter: `subscription_v_` (`WsSessionRouter.cpp:473`);
- UrlRouter: its maps (`UrlRouter.cpp:280`);
- WsSessionTable: `next_id_`, `session_map_` (`Webserver.cpp:1234`);
- WsSession: `output_buf_`, `outbound_q_` (`Webserver.cpp:1121`).

Retiring those printers (`issues/16`) needs their caller to hold the locks
for the whole print instead, or to print where nothing mutates. Find where
the server's snapshot is printed (introspect's publish path) and decide
which, and document the rule.

**Done when:** the caller of a server print holds what it needs, and no
websock printer takes a lock.
