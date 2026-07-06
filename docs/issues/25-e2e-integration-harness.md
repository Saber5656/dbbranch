# 25 — End-to-end integration test harness

## Title
Build the macOS E2E harness proving the full branch/switch/proxy workflow

## Summary
Create an end-to-end test harness (run on macOS CI with a real PostgreSQL and an
APFS volume) that exercises the whole product: init, create, switch, proxy
connectivity via a real client, isolation, hook-driven switching, crash recovery,
and idle-stop.

## Context
Whole-product validation strategy
([DESIGN §15 known unknowns](../DESIGN.md#15-known-unknowns-may-spawn-issues-during-implementation)).

## Scope
- In: E2E test suite + fixtures + CI job; a real `psql`-based connectivity check.
- Out: unit/integration tests owned by individual issues (they must exist too).

## Detailed Requirements
- Harness setup: create a scratch APFS volume or a directory known to be APFS
  (macOS CI default disk is APFS); use `$DBBRANCH_HOME` to isolate state; require
  a real PostgreSQL (matching the fixtures' `postgres_version`).
- Scenarios (each an assertion-backed test):
  1. **Init + URL**: `init --from-empty`; `psql "$(dbbranch url)"` connects.
  2. **Isolation**: switch A, write marker row; switch B (fresh from base), assert
     no marker; switch A, assert marker persists. Connection string constant
     throughout.
  3. **Instant clone**: seed base to ≥1 GB; time a `create`/`switch`; assert it
     completes in well under a second (CoW) — see [issue 08](08-cow-clone-engine.md).
  4. **Drain on switch**: hold an open `psql` session on A; switch to B; assert
     the open session still sees A while a new session sees B.
  5. **Hook flow**: install hook; `git switch`; assert active branch changed
     without git hanging/failing; detached HEAD is a no-op.
  6. **Crash recovery**: `kill -9` the daemon (and/or an instance); restart;
     assert reconciliation yields a consistent, serving state.
  7. **Idle stop + gc**: short `idle_timeout`; assert non-active idle instance
     stops; plant an orphan dir; assert `gc` removes it and never the base/active.
  8. **Security**: assert no instance TCP port is open and the proxy is
     loopback-only ([issue 20](20-security-loopback-perms-hba.md)); run the
     malicious-repo fixture and assert inert ([issue 21](21-security-untrusted-repo-config.md)).
- CI job `e2e` on `macos-latest`, gated but required for release; install
  PostgreSQL via Homebrew in the job.

## Acceptance Criteria
- All scenarios pass on macOS CI.
- The suite is deterministic (no flakiness from timing where avoidable; uses
  readiness polling not sleeps).
- A single command (`make e2e`) runs the whole suite locally.

## Validation
- The suite is itself the validation; it must be green in CI before v1 release.

## Dependencies
- All functional issues: 06, 07, 08, 09, 10, 11, 12, 13, 14, 15, 16, 17, 20, 21.

## Non-goals
- Cross-platform E2E (v2); performance benchmarking beyond the clone-time
  assertion.

## Design References
- [DESIGN §13](../DESIGN.md#13-failure-modes--edge-cases-cross-cutting), [§15](../DESIGN.md#15-known-unknowns-may-spawn-issues-during-implementation)
