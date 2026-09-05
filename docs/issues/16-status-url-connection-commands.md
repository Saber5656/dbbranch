# 16 — `status`, `url`, and `connection` commands

## Title
Implement `dbbranch status`, `dbbranch url`, and `dbbranch connection`

## Summary
Implement the read-only reporting commands that show project/branch state and
emit the stable connection string the app uses.

## Context
The stable connection string is the product's core UX
([ADR-002](../decisions/ADR-002-fixed-port-l4-proxy.md),
[DESIGN §12](../DESIGN.md#12-cli-surface-issue-map)).

## Scope
- In: `status`, `url`, `connection` CLI commands + their rendering of the daemon
  `status` result (issue 13).
- Out: the daemon `status` handler (issue 13) — this issue consumes it.

## Detailed Requirements
- `dbbranch url`: print exactly the stable connection URL and nothing else
  (script-friendly), e.g.
  `postgresql://<user>@127.0.0.1:<proxy_port>/<database>`. The user is the local
  OS user (trust auth, no password). No trailing newline noise beyond one `\n`.
- `dbbranch connection`: like `url` but may also print `PGHOST/PGPORT/PGDATABASE`
  export lines with `--export` for shell eval.
- `dbbranch status`: human-readable table (and `--json`) showing:
  - project root, project-id, PostgreSQL version, dbbranch home volume (APFS ✓).
  - proxy port, listen addr, connection string, active branch.
  - per-branch: name, state, createdFrom, last active, logical disk (CoW note).
  - health flags: daemon up, base present, any `error`-state branches,
    version/volume mismatches (defer detail to `doctor`, issue 19).
- If the project is not initialized, `status` prints a clear "run `dbbranch init`"
  message and exits non-zero; `url` errors similarly.
- Never print secrets (there are none by default; assert with issue 22 rules).

## Acceptance Criteria
- `url` output is a single valid connection string usable directly as
  `DATABASE_URL` and is invariant across switches.
- `status` (text + `--json`) reflects live state accurately.
- Uninitialized project yields actionable messages, non-zero exit.

## Validation
- Integration: `export DATABASE_URL=$(dbbranch url)` then `psql "$DATABASE_URL"`
  connects to the active branch; switch branches and confirm the same URL now
  reaches the new branch.
- Golden tests for `status --json`.

## Dependencies
- 12, 13.

## Non-goals
- Deep diagnostics (issue 19).

## Design References
- [DESIGN §12](../DESIGN.md#12-cli-surface-issue-map), [§1.1](../DESIGN.md#11-primary-user-story)
