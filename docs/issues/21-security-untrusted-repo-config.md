# 21 — Security: untrusted repo config hardening & fuzzing

## Title
Harden parsing of untrusted `.dbbranch/config.toml`/`seed.sql` against abuse

## Summary
Treat the repo-committed `.dbbranch/config.toml` and `seed.sql` as attacker
input (they ship in cloned repos) and prove, with negative and fuzz tests, that
they cannot cause arbitrary binary/command execution, path escape, network
exposure, or resource abuse.

## Context
Malicious-repo is the top abuse case
([DESIGN §11.2 case 1](../DESIGN.md#112-abuse-cases--mitigations)). This issue
adds adversarial coverage on top of the parser (issue 02).

## Scope
- In: adversarial validation rules, negative tests, fuzz harness, an explicit
  "capabilities the repo config CANNOT grant" test matrix.
- Out: the happy-path parser (issue 02); binary detection (issue 05).

## Detailed Requirements
- Explicit invariants enforced & tested (each a failing test if violated):
  1. No config field selects, names, or influences which binary is executed
     (PostgreSQL path comes only from detection/trusted config — issue 05).
  2. `sql_path` cannot escape the repo tree: reject absolute paths, `..`
     segments, and symlinks resolving outside the repo (`filepath.EvalSymlinks`
     + prefix check).
  3. `proxy_port` outside `1024..65535` rejected; ports cannot bind non-loopback
     (issue 20 enforces bind side too).
  4. `postgres_version` cannot be a path/command; must match `^[0-9]{1,2}$`.
  5. Unknown keys rejected (no silent behavior injection).
  6. Numeric fields have bounds (`max_instances 1..64`); oversized/negative
     rejected.
  7. Seed SQL is executed **only** on explicit `init`/`base import`/`base
     refresh`, **never** implicitly on `switch` or hook.
  8. Branch names from git/config are encoded (issue 04) so they cannot traverse
     directories.
- Fuzz harness: `go test -fuzz` over the TOML loader (issue 02) and the
  `sql_path` resolver; seed corpus with traversal, null bytes, huge values,
  duplicate keys, unicode, symlink tricks.
- Document the guarantees in `SECURITY.md` (issue 24) and link back here.

## Acceptance Criteria
- Every invariant has a negative test that fails closed.
- Fuzzing the loader/resolver for a bounded time finds no panic and no path
  escape (CI runs a short fuzz; longer fuzz documented).
- A crafted malicious `.dbbranch/config.toml`/`seed.sql` in a test fixture cannot
  execute a planted binary, read/write outside the repo/home, or open a
  non-loopback port.

## Validation
- Negative + fuzz tests in CI (macOS). Include a "malicious repo" fixture and an
  end-to-end assertion that running dbbranch in it is inert/safe.

## Dependencies
- 02, 04, 05, 20.

## Non-goals
- Sandboxing the user's own PostgreSQL binaries (trusted toolchain, ADR-003).

## Design References
- [DESIGN §11.2](../DESIGN.md#112-abuse-cases--mitigations), [§5.4](../DESIGN.md#54-configtoml-committed-untrusted-schema)
