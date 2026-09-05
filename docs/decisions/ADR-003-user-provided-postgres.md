# ADR-003: Use the user's installed PostgreSQL; never bundle or download server binaries

- Status: Accepted
- Date: 2026-07-06
- Deciders: Project owner (human), Fable (design)

## Context

dbbranch needs `initdb`, `postgres`, `pg_ctl`, `pg_isready`, `psql`. Options:

- **A. Detect and use the user's existing PostgreSQL** (Homebrew, etc.).
- **B. dbbranch downloads/embeds a pinned PostgreSQL build.**
- **C. Both** (prefer existing, download if missing).

Human decision (recorded): **A — use the user's existing PostgreSQL.**

## Decision

dbbranch locates the user's installed PostgreSQL binaries (trusted local config
override → `PATH` → known Homebrew paths), resolves them to absolute paths,
verifies the major version, and uses them. It never downloads or bundles a
PostgreSQL server.

## Rationale

- No supply-chain surface from fetching/verifying third-party server binaries at
  runtime (option B's signing/provenance/update burden is avoided).
- Matches the user's actual dev PostgreSQL (extensions, version, locale),
  reducing "works on my branch but not in prod" drift.
- Simpler, smaller distribution; dbbranch itself is one Go binary.

## Consequences

- Startup depends on a compatible PostgreSQL being installed; `doctor`/`init`
  must detect and give actionable remediation when absent or mismatched.
- Base and clones are pinned to one PostgreSQL **major** version; a user major
  upgrade invalidates existing base clusters (documented manual `pg_upgrade`
  path; automation deferred to v2) — [DESIGN §8.3](../DESIGN.md#83-version-consistency-invariant).
- Detection must be hardened against PATH hijacking (absolute resolve, pin in
  state) — [DESIGN §11.2](../DESIGN.md#112-abuse-cases--mitigations).
