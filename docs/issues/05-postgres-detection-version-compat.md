# 05 — PostgreSQL binary detection & version compatibility

## Title
Detect PostgreSQL binaries, resolve absolute paths, and enforce version compat

## Summary
Implement `internal/pgfind`: locate `initdb`, `postgres`, `pg_ctl`, `pg_isready`,
`psql`; resolve to absolute paths; parse the version; and enforce that the
detected major version matches the project's required `postgres_version`. Also
load the trusted local config that may override the bin dir.

## Context
dbbranch uses the user's installed PostgreSQL
([ADR-003](../decisions/ADR-003-user-provided-postgres.md),
[DESIGN §8.1](../DESIGN.md#81-detection-internalpgfind-issue-05)). Detection must
be hardened against PATH hijacking.

## Scope
- In: detection order, absolute/realpath resolution, version parse, compat check,
  same-install validation, trusted local config load (`~/.dbbranch/config.toml`).
- Out: initdb/start (issue 06/07).

## Detailed Requirements
- `LocalConfig` struct + `LoadLocalConfig()` for `~/.dbbranch/config.toml`
  ([DESIGN §5.5](../DESIGN.md#55-localglobal-trusted-config-dbbranchconfigtoml-schema)):
  fields `Version`, `PostgresBinDir`, `DaemonIdleShutdown`, `LogLevel`. This file
  is **trusted**; only it may set an absolute `PostgresBinDir`.
- `Detect(requiredMajor int, trustedBinDir string) (*Toolchain, error)` search
  order ([DESIGN §8.1](../DESIGN.md#81-detection-internalpgfind-issue-05)):
  1. `trustedBinDir` (from trusted config) if non-empty.
  2. `PATH` (via `exec.LookPath`).
  3. Known Homebrew dirs: `/opt/homebrew/opt/postgresql@<major>/bin`,
     `/usr/local/opt/postgresql@<major>/bin`.
- `Toolchain{ BinDir, Initdb, Postgres, PgCtl, PgIsready, Psql string,
  MajorVersion int, FullVersion string }`. All paths `realpath`-resolved absolute.
- Version: run `<postgres> --version`; parse `PostgreSQL) (\d+)\.(\d+)`; the
  **major** must equal `requiredMajor` or return `ErrVersionMismatch` with
  remediation text (install `postgresql@<major>`).
- **Same-install invariant**: all five binaries must resolve to the same
  `BinDir`; otherwise `ErrMixedInstall`.
- Never execute a binary path that came from untrusted repo config (it can't —
  repo config has no such field; assert this in code comments/tests).

## Acceptance Criteria
- On a machine with matching PostgreSQL, `Detect` returns a fully-populated,
  absolute-path `Toolchain` with correct major version.
- Missing binaries → actionable `ErrNotFound` naming what to install.
- Major mismatch → `ErrVersionMismatch`.
- Trusted `PostgresBinDir` overrides PATH.
- Mixed installs are rejected.

## Validation
- Unit tests with a fake bin dir of shell-script stubs emulating
  `postgres --version` output for several versions.
- Test PATH-hijack resistance: a `postgres` earlier in PATH than the required one
  yields mismatch, not silent use.

## Dependencies
- 01, 02 (for `postgres_version` value).

## Non-goals
- Running initdb/postgres (issues 06/07); upgrade automation (v2).

## Design References
- [ADR-003](../decisions/ADR-003-user-provided-postgres.md)
- [DESIGN §8.1](../DESIGN.md#81-detection-internalpgfind-issue-05), [§8.3](../DESIGN.md#83-version-consistency-invariant)
