# 10 — Loopback TCP proxy with connection draining

## Title
Implement the L4 loopback proxy that routes to the active branch and drains

## Summary
Implement `internal/proxy`: a per-project TCP listener on `127.0.0.1:<port>` that
byte-splices each new connection to the currently active branch's Unix socket,
selecting the target at accept time and letting existing connections drain on
target change.

## Context
Stable connection string via a fixed port
([ADR-002](../decisions/ADR-002-fixed-port-l4-proxy.md),
[DESIGN §10](../DESIGN.md#10-proxy-internalproxy-issue-10)).

## Scope
- In: listener, target selection, byte splice, draining, target switch API,
  connection accounting, loopback-only bind assertion.
- Out: deciding the active branch (issue 11); daemon wiring (issue 12).

## Detailed Requirements
- `Proxy` type with:
  - `New(listenAddr string) (*Proxy, error)`: **assert** `listenAddr` host is a
    loopback IP (`127.0.0.0/8` or `::1`); reject otherwise (issue 20). Bind fails
    on port-in-use → typed error.
  - `SetTarget(unixSocketPath string)`: atomically update the current target
    (the `.s.PGSQL.5432` path of the active branch). Affects only future accepts.
  - `Serve()` / `Close()`.
  - `ActiveConns(targetPath string) int`: number of live proxied connections to a
    given target (used by idle-stop/draining, issue 15).
- On `accept`: snapshot the current target; if none/empty, close the client
  connection immediately (or after a short readable error) — surface "no active
  branch". Else `net.Dial("unix", target)`, then bidirectional `io.Copy` in two
  goroutines; close both when either side EOFs; decrement the per-target counter.
- Target change never disturbs in-flight connections (they keep their dialed
  backend). New connections go to the new target.
- Graceful `Close`: stop accepting, then wait (bounded) for in-flight copies or
  force-close after a deadline.

## Acceptance Criteria
- A client connecting to `127.0.0.1:<port>` reaches the active branch's
  PostgreSQL (verified with `psql "host=127.0.0.1 port=<port> dbname=app"`).
- After `SetTarget` to another branch, **new** `psql` connections hit the new DB
  while an already-open session keeps talking to the old DB until it closes.
- Binding a non-loopback address is refused.
- `ActiveConns` correctly counts open proxied connections.
- Port-in-use yields an actionable error.

## Validation
- Integration on macOS CI: two branches with distinguishable data (e.g. a marker
  row); open a long-lived `psql` on branch A, switch target to B, confirm the old
  session still sees A and a new session sees B.
- Concurrency/race test under `-race` with many simultaneous connections.

## Dependencies
- 01, 07 (a running instance to target).

## Non-goals
- Protocol-aware routing/auth (v2); choosing the active branch (issue 11).

## Design References
- [DESIGN §10](../DESIGN.md#10-proxy-internalproxy-issue-10), [ADR-002](../decisions/ADR-002-fixed-port-l4-proxy.md)
