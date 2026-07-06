# 20 — Security: loopback-only binding, filesystem perms, pg_hba generation

## Title
Enforce loopback-only binding, 0700/0600 perms, and minimal same-user pg_hba

## Summary
Implement the network- and filesystem-hardening controls that keep dbbranch a
local-only tool: assert loopback-only TCP binds, create all paths with tight
perms, and generate a minimal `pg_hba.conf`/`postgresql.auto.conf` that only
permits same-user local socket access.

## Context
[DESIGN §11 security model](../DESIGN.md#11-security-model),
[ADR-006](../decisions/ADR-006-unix-socket-instances-loopback-proxy.md).

## Scope
- In: loopback-bind assertion, perms enforcement helpers, PostgreSQL auth/socket
  config generation, a self-test.
- Out: untrusted repo-config hardening (issue 21); secrets (issue 22).

## Detailed Requirements
- Loopback assertion (shared helper): before binding any TCP listener, verify the
  host resolves to a loopback address; refuse `0.0.0.0`, `::`, or any non-loopback
  IP. Used by the proxy (issue 10) and asserted in tests.
- Instance network config (used by issues 06/07):
  - `postgresql.auto.conf`: `listen_addresses = ''` (no TCP),
    `unix_socket_directories = '<run/<branch>>'`, `unix_socket_permissions = 0700`.
  - `pg_hba.conf`: only `local all <osuser> trust` (and `local all all trust`
    scoped to the socket dir is acceptable since the dir is `0700`); no `host`
    lines. Rationale: the socket dir perms (`0700`, same user) are the real
    boundary; `trust` is safe there and avoids storing/handling a password.
- Perms helpers (extend issue 04): create dirs `0700`, files `0600`; a
  `HardenPath` that `chmod`s existing dbbranch-owned paths to the intended perms
  and refuses symlinked components.
- Security self-test surfaced in `doctor` (issue 19): assert no instance TCP port
  is open, the proxy is loopback-only, and state/socket perms are correct.

## Acceptance Criteria
- No PostgreSQL instance opens any TCP port (verified via `lsof -iTCP`/`ss` in
  tests); only the proxy listens, on `127.0.0.1`.
- Any attempt to bind a non-loopback address is refused.
- All created dirs are `0700`, files `0600`.
- Generated `pg_hba.conf` contains no `host` lines.

## Validation
- Integration on macOS CI: bring up an instance + proxy, scan open sockets,
  assert loopback-only and no instance TCP; perms assertions on the tree.
- Unit tests for the loopback assertion (table of allowed/denied addresses).

## Dependencies
- 04, 06, 07, 10.

## Non-goals
- Password auth (intentionally avoided in v1 via socket+trust).

## Design References
- [DESIGN §11.1](../DESIGN.md#111-trust-boundaries), [§11.3](../DESIGN.md#113-secure-defaults), [ADR-006](../decisions/ADR-006-unix-socket-instances-loopback-proxy.md)
