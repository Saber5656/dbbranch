# 19 — Structured logging, error taxonomy & `dbbranch doctor`

## Title
Implement structured logging, a typed error taxonomy, and `dbbranch doctor`

## Summary
Provide consistent structured logging (daemon + CLI), a shared typed error
taxonomy with stable codes and actionable messages, and a `dbbranch doctor`
command that diagnoses the environment.

## Context
Cross-cutting quality: clear diagnostics for the failure modes in
[DESIGN §13](../DESIGN.md#13-failure-modes--edge-cases-cross-cutting).

## Scope
- In: logging setup, error type/codes, `doctor` checks.
- Out: the operations that raise the errors (their own issues).

## Detailed Requirements
- Logging: use `log/slog`. Daemon logs to `~/.dbbranch/daemon.log` (level from
  trusted config `log_level`). CLI logs to stderr; `--verbose` raises level,
  `--quiet` lowers. Never log secrets (issue 22); provide a redaction helper.
- Error taxonomy `internal/dberr`: `Error{Code, Message, Hint, Cause}` with stable
  `Code` constants matching the daemon RPC codes (issue 13). `Error()` renders
  `message (hint)`; `--verbose` also prints the cause chain.
- `dbbranch doctor`:
  - Checks (each PASS/WARN/FAIL with remediation):
    - dbbranch home exists, perms `0700`, on an **APFS** volume.
    - PostgreSQL detected, absolute paths, major matches `postgres_version`.
    - Daemon reachable (or startable).
    - Proxy port bindable / not conflicting.
    - Base cluster present and startable (dry check).
    - No `error`-state branches; no orphan pgdata dirs.
    - Git available; repo detected.
  - `--json` output; non-zero exit if any FAIL.

## Acceptance Criteria
- Logs are structured, leveled, and secret-free.
- Errors carry stable codes and actionable hints; the same code appears in RPC
  and CLI output.
- `doctor` correctly reports PASS on a healthy setup and FAIL with remediation on
  each seeded fault (non-APFS home, missing PostgreSQL, version mismatch, port in
  use, orphan dir).

## Validation
- Unit tests for error rendering/redaction.
- Integration on macOS CI: `doctor` on a healthy project (all PASS) and with each
  injected fault.

## Dependencies
- 01, 04, 05, 08 (APFS check), 12.

## Non-goals
- Auto-repair (doctor reports; `gc`/manual steps fix).

## Design References
- [DESIGN §13](../DESIGN.md#13-failure-modes--edge-cases-cross-cutting)
