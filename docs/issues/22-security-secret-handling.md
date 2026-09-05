# 22 — Security: secret handling & redaction

## Title
Ensure secrets (if any) are generated, stored, and logged safely

## Summary
Establish secret-handling rules: v1 uses passwordless local trust auth, but if a
password is ever generated (optional/opt-in or future), it must be random,
stored `0600`, never logged, and redacted everywhere; and connection strings/logs
must never leak credentials.

## Context
[DESIGN §11.2 case 5](../DESIGN.md#112-abuse-cases--mitigations),
[§11.3](../DESIGN.md#113-secure-defaults).

## Scope
- In: redaction helper, secret storage rules, connection-string handling,
  ensuring state/logs are outside the repo and not committed.
- Out: implementing password auth (v1 avoids it); TLS (v2).

## Detailed Requirements
- Default posture: no DB password (socket + `trust`, issue 20). The connection
  URL therefore contains no secret; document this.
- If an optional password mode is added (guard behind a flag; may be a stub in
  v1): generate via a CSPRNG (`crypto/rand`), ≥ 24 chars, store in
  `~/.dbbranch/projects/<id>/secret` mode `0600`, never in repo, never in
  `state.json` in plaintext (store a reference/handle).
- Redaction helper `redact(s string) string` used by all logging/error paths;
  patterns for `password=`, URLs with `user:pass@`, etc. Unit-tested.
- `.gitignore` guidance emitted by `init`: ensure no dbbranch local artifacts
  land in the repo; assert dbbranch never writes secrets under the repo tree.
- Ensure `--json` outputs and `status`/`url` never include a password even if one
  exists (offer a separate explicit `--show-password` if ever needed).

## Acceptance Criteria
- Default flow exposes no secret anywhere; `url` has no credentials.
- Redaction helper masks known secret patterns; verified by tests over log/error
  samples.
- Any secret file created is `0600` and outside the repo.
- No secret appears in logs, `status`, `--json`, or error output (test scans
  captured output for planted secrets).

## Validation
- Unit tests for redaction; integration test that greps all CLI/daemon output and
  log files for a planted secret sentinel and finds none.

## Dependencies
- 19 (logging), 20 (auth model).

## Non-goals
- Implementing full password/TLS auth (v2).

## Design References
- [DESIGN §11.2](../DESIGN.md#112-abuse-cases--mitigations), [§11.3](../DESIGN.md#113-secure-defaults)
