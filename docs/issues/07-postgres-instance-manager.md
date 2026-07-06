# 07 — PostgreSQL instance manager (single instance lifecycle)

## Title
Start, stop, and health-check a single PostgreSQL instance on a private socket

## Summary
Implement `internal/pg`: functions to `initdb`, start, wait-for-ready, stop, and
inspect a single PostgreSQL instance bound to a per-branch Unix socket, with log
capture and pid tracking. No TCP is ever exposed by an instance.

## Context
Each branch is served by its own instance
([DESIGN §8.2](../DESIGN.md#82-instance-manager-internalpg-issue-07),
[ADR-006](../decisions/ADR-006-unix-socket-instances-loopback-proxy.md)).

## Scope
- In: `Initdb`, `Start`, `WaitReady`, `Stop`, `IsRunning`, `Exec` (psql helper),
  config-file generation for an instance, log capture.
- Out: cloning (issue 08), proxy (issue 10), orchestration/state (daemon).

## Detailed Requirements
- `Initdb(tc Toolchain, pgdata string) error`:
  `initdb -D <pgdata> --encoding=UTF8 --locale=C -A trust`; then write
  `postgresql.auto.conf` and `pg_hba.conf` per issue 20 (helper may live in 20).
- `Start(tc, pgdata, socketDir, logPath string) (pid int, err error)`:
  ensure `socketDir` exists `0700`; remove stale `postmaster.pid` if the recorded
  pid is dead; run
  `pg_ctl -D <pgdata> -l <logPath> -o "-k <socketDir> -c listen_addresses=''" -w start`.
  Return the postmaster pid (read from `<pgdata>/postmaster.pid` line 1).
- `WaitReady(tc, socketDir string, timeout)`:
  poll `pg_isready -h <socketDir> -p 5432` until ready or timeout (default 30s);
  on timeout return an error including the tail of `logPath`.
- `Stop(tc, pgdata string, mode="fast", timeout)`:
  `pg_ctl -D <pgdata> -m fast -w stop`; on timeout escalate to `-m immediate`,
  then error if still alive.
- `IsRunning(pgdata string) (bool, pid int)`: parse `postmaster.pid`, verify the
  pid is alive and its start time / data dir matches (defense against pid reuse).
- `Exec(tc, socketDir, database, sql string) error` and
  `ExecFile(tc, socketDir, database, sqlPath string) error`: run via
  `psql -h <socketDir> -p 5432 -d <database> -v ON_ERROR_STOP=1`.
- All commands run with an explicit env (do not inherit `PGHOST`, `PGPORT`,
  `PGDATA`, `PGUSER` from the user's shell that could redirect to a wrong server;
  set them explicitly).

## Acceptance Criteria
- `Initdb`+`Start`+`WaitReady` brings up a real instance reachable only via its
  Unix socket; `psql` over the socket works; no TCP port is opened (verify with
  `lsof`/`netstat` in test).
- `Stop` cleanly shuts down; `IsRunning` reflects true/false accurately including
  after a kill -9 (stale pid detected).
- Ready-timeout returns an error containing log tail.
- Env isolation: a hostile `PGHOST` in the environment does not affect operations.

## Validation
- Integration tests on macOS CI with real PostgreSQL: full lifecycle, stale-pid
  recovery, timeout path (point at a broken pgdata).

## Dependencies
- 01, 04, 05; 20 for the config/hba generation helper.

## Non-goals
- Managing many instances / scheduling (daemon, issue 12); cloning (issue 08).

## Design References
- [DESIGN §8.2](../DESIGN.md#82-instance-manager-internalpg-issue-07)
- [ADR-006](../decisions/ADR-006-unix-socket-instances-loopback-proxy.md)
