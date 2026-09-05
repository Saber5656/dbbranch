# ADR-006: Instances on Unix sockets; only the proxy binds loopback TCP

- Status: Accepted
- Date: 2026-07-06
- Deciders: Fable (design), pending human ratification

## Context

Multiple PostgreSQL instances plus a proxy plus a daemon control plane all run on
one developer machine. Each listener is attack surface. We must minimize network
exposure of developer databases (a common, serious foot-gun) while keeping a
stable TCP endpoint for the app ([ADR-002](ADR-002-fixed-port-l4-proxy.md)).

## Decision

- **PostgreSQL instances** listen on **Unix domain sockets only**
  (`listen_addresses = ''`), in per-branch directories with mode `0700`.
- **The proxy** is the only component that binds TCP, and only on
  `127.0.0.1:<proxy_port>` (never `0.0.0.0`/non-loopback).
- **The daemon control plane** is a Unix domain socket (`0600`), not TCP.

## Rationale

- Instances carry no TCP surface at all; they are reachable only by same-user
  processes with filesystem access to the `0700` socket dir.
- A single, loopback-only TCP endpoint is the minimum needed for the app.
- Filesystem permissions authenticate the control plane; no network auth to get
  wrong.

## Consequences

- The proxy bridges TCP↔Unix socket.
- A startup assertion must reject any attempt to bind a non-loopback address
  (defense in depth) — [issue 20](../issues/20-security-loopback-perms-hba.md).
- `pg_hba.conf` only needs to permit local same-user socket auth (`trust`),
  keeping the auth surface minimal — [DESIGN §11.3](../DESIGN.md#113-secure-defaults).
