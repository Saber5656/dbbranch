# 06 — `dbbranch init`: base cluster creation & project bootstrap

## Title
Implement `dbbranch init` to bootstrap a project and its base PostgreSQL cluster

## Summary
Implement `dbbranch init`: detect environment, create the project home, `initdb`
the base cluster with secure defaults, create the app database, optionally seed
it, write `project.json` and initial `state.json`, and optionally install the
Git hook.

## Context
`init` is the entry point that makes a repo dbbranch-managed. See
[DESIGN §12](../DESIGN.md#12-cli-surface-issue-map) and
[DESIGN §8.2](../DESIGN.md#82-instance-manager-internalpg-issue-07).

## Scope
- In: the `init` command orchestration, base `initdb`, app-DB creation, optional
  seed, state bootstrap, `--install-hook` passthrough, preflight checks.
- Out: switching/cloning (issues 09/11); daemon RPC (may run in-process for init
  or delegate — see Requirements).

## Detailed Requirements
- Flags: `--from-empty` (default), `--from-dump <file>` (seed via a SQL dump),
  `--install-hook`, `--postgres-version <n>` (overrides/creates committed config),
  `--force` (re-init; refuse if branches exist unless forced).
- Preflight (fail closed with remediation):
  - Repo root detected (issue 04).
  - dbbranch home volume is APFS (issue 08 helper).
  - `RepoConfig` present or created interactively/from flags (issue 02); if
    absent, write a `.dbbranch/config.toml` with provided/default values.
  - Toolchain detected & version-compatible (issue 05).
- Actions:
  1. `EnsureDirs` for the project (issue 04), `0700`.
  2. `initdb -D <base/pgdata> --encoding=UTF8 --locale=C -A trust` (issue 07
     wrapper). Write hardened `postgresql.auto.conf` (`listen_addresses=''`,
     `unix_socket_directories`, `unix_socket_permissions=0700`) and a minimal
     `pg_hba.conf` (local socket, same user, `trust`) — see issue 20.
  3. Start base temporarily on a private socket, `CREATE DATABASE <database>`,
     run seed if `--from-dump` (psql < dump) — abort & clean up on seed error.
  4. Stop base (quiesced; it is a clone source, [ADR-005](../decisions/ADR-005-clone-consistency-strategy.md)).
  5. Allocate `proxy_port` (use configured; else pick a free loopback port and
     record it in state).
  6. Write `project.json` and `state.json` (schemaVersion 1), base branch record
     in `stopped` state.
  7. If `--install-hook`, invoke hook install (issue 17).
- Idempotency: re-running without `--force` on an initialized project prints
  current status and exits 0; with `--force` it refuses if branch clones exist
  (must `gc`/`delete` first).

## Acceptance Criteria
- Fresh repo: `dbbranch init` creates base cluster + app DB + state; `status`
  then shows base `stopped`, a proxy port, and a valid connection string.
- `--from-dump` loads the dump; a broken dump aborts init and leaves no
  half-built base.
- Preflight failures (non-APFS, missing/mismatched PostgreSQL, not a repo) print
  actionable errors and exit non-zero.
- Perms of created dirs/files are `0700`/`0600`.

## Validation
- E2E on macOS CI with a real PostgreSQL: init empty, init from a small dump,
  and each preflight failure path.
- Verify `initdb` auth/socket config matches issue 20 expectations.

## Dependencies
- 02, 03, 04, 05, 07, 20 (config hardening); 17 optional for `--install-hook`.

## Non-goals
- Branch cloning/switching (09/11); base re-import after init (issue 18).

## Design References
- [DESIGN §8.2](../DESIGN.md#82-instance-manager-internalpg-issue-07), [§9.2](../DESIGN.md#92-consistency-adr-005), [§11](../DESIGN.md#11-security-model)
