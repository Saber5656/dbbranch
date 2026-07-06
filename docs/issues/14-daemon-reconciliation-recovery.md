# 14 — Daemon reconciliation & crash recovery

## Title
Reconcile persisted state with reality on daemon startup

## Summary
On daemon startup, reconcile each project's `state.json` with actual processes
and files: detect dead/orphan instances, clear stale pids and sockets, and repair
the active pointer, so a crashed daemon recovers to a consistent state.

## Context
[DESIGN §7.5 reconciliation](../DESIGN.md#75-reconciliation-crash-recovery).

## Scope
- In: startup reconciliation pass per project; orphan detection; state repair.
- Out: user-invoked `gc` (issue 15) — but they share a scan helper.

## Detailed Requirements
- For each known project (projects registry seeded from `~/.dbbranch/projects/*`):
  1. Load `state.json` (skip with a logged warning if `ErrStateTooNew`).
  2. For each branch with a recorded `pid`: verify via
     `pg.IsRunning(pgdata)` (pid alive **and** matches this pgdata's
     `postmaster.pid`). If not running → set `stopped`, clear pid.
  3. Remove stale socket files under `run/<branch>/` for non-running branches.
  4. Detect orphan `branches/*/pgdata` dirs with no state record and record them
     as gc candidates (do not auto-delete here).
  5. If `activeBranch` is set but its instance isn't running, either restart it
     (if its clone is intact) or clear the active pointer and log; the proxy then
     reports "no active branch" until the next switch.
  6. Rebuild proxy listeners for projects that had an active branch (bind the
     recorded port; on conflict, log and mark degraded).
- Reconciliation must be safe to run repeatedly (idempotent).

## Acceptance Criteria
- After `kill -9` of the daemon (leaving instances running or dead), a fresh
  daemon start yields a consistent state: live instances re-adopted, dead ones
  marked stopped, stale sockets removed.
- Orphan pgdata dirs are surfaced (via `status`/`gc`) not silently deleted.
- A previously active branch is restored to serving or cleanly reported inactive.

## Validation
- Integration on macOS CI: simulate crash scenarios (kill daemon; kill an
  instance; leave a stale socket/pid; create an orphan dir) and assert the
  post-start state.

## Dependencies
- 03, 04, 07, 10, 12.

## Non-goals
- Deleting orphans (issue 15).

## Design References
- [DESIGN §7.5](../DESIGN.md#75-reconciliation-crash-recovery)
