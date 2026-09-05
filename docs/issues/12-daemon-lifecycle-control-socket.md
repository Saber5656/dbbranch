# 12 — Daemon lifecycle & control socket

## Title
Implement `dbbranchd` lifecycle, single-instance lock, and the control socket RPC

## Summary
Implement the `dbbranchd` daemon: a single per-user process that owns state,
instances, and proxies; listens on a `0600` Unix control socket speaking
newline-delimited JSON RPC; supports auto-start from the CLI; and self-shuts-down
when idle.

## Context
[DESIGN §7](../DESIGN.md#7-daemon-dbbranchd),
[ADR-006](../decisions/ADR-006-unix-socket-instances-loopback-proxy.md).

## Scope
- In: daemon main loop, control socket server, request framing/dispatch, single
  instance lock, auto-start from CLI, idle self-shutdown, per-project registry
  and locks.
- Out: the handler bodies for each method (issue 13 wires them to 06/09/11/etc.);
  reconciliation (issue 14); idle instance stop (issue 15).

## Detailed Requirements
- `cmd/dbbranchd`: start the daemon; acquire single-instance lock via `flock` on
  `~/.dbbranch/daemon.pid`; if already held by a live process, exit 0.
- Control socket: listen on `~/.dbbranch/daemon.sock` (unlink stale socket first;
  create with `0600`, parent dir `0700`). Reject if perms/owner unexpected.
- RPC framing per [DESIGN §7.2](../DESIGN.md#72-control-protocol): one JSON
  request per line, one JSON response per line; fields `id`, `method`, `params`;
  responses `ok`+`result` or `ok:false`+`error{code,message}`.
- Dispatch table registering method names (`ping, init, status, list, switch,
  create, delete, reset, stop, gc, baseImport, shutdown`). Unknown method →
  error `code:"unknown_method"`. Handlers themselves are issue 13 (register stubs
  here returning `not_implemented` except `ping`, `shutdown`).
- Every request carries `projectRoot`; the daemon re-canonicalizes it and derives
  the project-id itself (never trusts a client-supplied id). A per-project
  registry lazily loads state and holds a per-project mutex.
- CLI client helper `internal/daemon/client`: connect to the socket; if connect
  refused, spawn `dbbranchd` detached (double-fork), wait for `ping` (≤5s), retry.
- Idle self-shutdown: after `daemon_idle_shutdown` (default 30m; `0` disables)
  with zero running instances across all projects, exit cleanly (stop proxies).
- Signal handling: on SIGTERM/SIGINT, stop all instances (`pg_ctl -m fast`),
  close proxies/sockets, release locks, exit 0.

## Acceptance Criteria
- Starting `dbbranchd` twice results in one running daemon.
- `ping` round-trips over the socket; unknown methods error cleanly.
- CLI auto-starts the daemon on first use and connects.
- Socket is `0600`, parent `0700`; a wrong-perm socket is refused/recreated.
- SIGTERM stops all instances and exits cleanly.
- Idle daemon self-exits after the configured timeout.

## Validation
- Integration on macOS CI: auto-start, `ping`, concurrent connections,
  double-start, SIGTERM cleanup, idle shutdown (short timeout).

## Dependencies
- 01, 03, 04, 05, 10 (proxy owned by daemon).

## Non-goals
- Handler bodies (13); reconciliation (14); idle instance stop (15).

## Design References
- [DESIGN §7](../DESIGN.md#7-daemon-dbbranchd)
