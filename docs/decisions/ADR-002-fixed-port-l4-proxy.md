# ADR-002: Fixed loopback port + L4 proxy for a stable connection string

- Status: Accepted
- Date: 2026-07-06
- Deciders: Project owner (human), Fable (design)

## Context

Each branch runs its own PostgreSQL instance on its own socket/port
([ADR-001](ADR-001-cow-clone-per-branch-instance.md)). The developer's app needs
a connection target. Options considered:

- **A. Fixed port + proxy**: dbbranch listens on one stable loopback port and
  routes to the active branch. App connection string is constant.
- **B. Per-branch port surfaced to the user**: app reconfigures each switch.
- **C. Fixed port, rebind the active instance to it (no proxy)**: switch rebinds;
  existing connections drop; port-conflict/startup-wait handling needed.

Human decision (recorded): **A — fixed port + proxy, connection string invariant.**

## Decision

Run an **L4 (byte-stream) TCP proxy** bound to `127.0.0.1:<proxy_port>` per
project. New connections are dialed to the **currently active** branch's Unix
socket. The proxy does not parse the PostgreSQL wire protocol in v1.

## Rationale

- Constant connection string = the core "zero app reconfiguration" UX (option B
  loses this).
- A proxy lets us **drain** old connections on switch rather than dropping them
  (option C drops in-flight transactions).
- L4-only keeps the proxy simple and low-attack-surface: it never sees
  credentials or query contents, just opaque bytes. Auth stays in PostgreSQL.

## Consequences

- Must implement a small, correct TCP splice with target selection at accept
  time and connection draining ([DESIGN §10](../DESIGN.md#10-proxy-internalproxy-issue-10)).
- Instances bind Unix sockets only; only the proxy binds TCP, and only on
  loopback ([ADR-006](ADR-006-unix-socket-instances-loopback-proxy.md)).
- Switch does not interrupt in-flight connections; they finish against the old
  instance, which stays alive until it has zero proxied connections.
- No protocol-level features (query routing, auth interception) in v1; deferrable.
