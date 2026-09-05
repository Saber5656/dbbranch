# 15 — Idle instance stop, instance caps, and `dbbranch gc`

## Title
Implement idle-stop, the max-instances cap, and garbage collection

## Summary
Bound resource usage: stop idle drained instances after a timeout, enforce a
concurrent-instance cap, and provide `dbbranch gc` to remove orphaned clones and
stopped instances.

## Context
Many branches ⇒ many instances ⇒ memory/disk pressure
([DESIGN §11.2 abuse case 7](../DESIGN.md#112-abuse-cases--mitigations),
[DESIGN §7](../DESIGN.md#7-daemon-dbbranchd)).

## Scope
- In: idle-stop timer loop, cap enforcement on start, `gc` command + operation.
- Out: reconciliation (issue 14, shares the scan helper).

## Detailed Requirements
- Idle-stop: a daemon timer (per project) periodically checks each `running`
  branch that is **not** the active branch and has **zero** proxied connections
  (`proxy.ActiveConns`) for ≥ `idle_timeout` (config, default 10m; `0` disables).
  Such instances are `pg_ctl -m fast` stopped → `stopped`. The active branch is
  never idle-stopped.
- Max-instances cap: before starting an instance, if running count ≥
  `max_instances` (default 5, config), stop the least-recently-active
  non-active running instance with zero connections; if none is eligible, return
  a typed `too_many_instances` error advising `gc`/raising the cap.
- `gc` operation + `dbbranch gc [--dry-run] [--yes]`:
  - Stop instances for branches not accessed within a threshold (optional flag).
  - Remove orphan `branches/*/pgdata` dirs with no state record (from issue 14
    detection).
  - Remove stopped branches on request (`--branches`), never the active branch or
    `base`.
  - `--dry-run` lists actions; without `--yes`, prompt for confirmation.
  - Report reclaimed logical space (note CoW-shared caveat).

## Acceptance Criteria
- An idle, non-active, connection-free instance is stopped after the timeout; the
  active one is not.
- Starting beyond the cap reclaims an eligible instance or errors clearly.
- `gc --dry-run` lists orphans/candidates without changing anything; `gc --yes`
  removes them; base and active branch are never removed.

## Validation
- Integration on macOS CI with a short `idle_timeout`: open then close a
  connection, assert stop after timeout; cap test with `max_instances=2`;
  gc test with a planted orphan dir.

## Dependencies
- 03, 07, 10, 12, 14.

## Non-goals
- Disk quotas (v2); cross-project global caps (v2).

## Design References
- [DESIGN §7](../DESIGN.md#7-daemon-dbbranchd), [§11.2](../DESIGN.md#112-abuse-cases--mitigations)
