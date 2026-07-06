# 13 — Daemon RPC handlers (wire CLI to operations)

## Title
Wire daemon RPC methods to the branch/switch/init operations

## Summary
Implement the daemon-side handler bodies that map each RPC method to the
operations from issues 06/09/11/15/18 and return structured results the CLI
renders. This is the integration layer between transport (issue 12) and logic.

## Context
[DESIGN §7.2](../DESIGN.md#72-control-protocol),
[DESIGN §12](../DESIGN.md#12-cli-surface-issue-map).

## Scope
- In: handler funcs for `init, status, list, switch, create, delete, reset, stop,
  gc, baseImport`; request param structs; result structs; error mapping.
- Out: the underlying operations (already in their issues); CLI rendering (each
  command issue owns its own output, but result schemas are defined here).

## Detailed Requirements
- Define request/response param structs per method (JSON-tagged), e.g.
  `SwitchParams{ProjectRoot, Branch, From string; IfManaged bool}`,
  `SwitchResult{ActiveBranch, ConnectionString string; Branches []BranchView}`.
- Each handler: acquire the per-project mutex (issue 12 registry), call the
  operation, translate typed operation errors into RPC `error{code,message}`
  with stable `code` values (`not_initialized`, `branch_exists`, `active_branch`,
  `version_mismatch`, `not_apfs`, `clone_unsupported`, `port_in_use`, ...).
- `status` returns the full project view: active branch, connection string,
  proxy port, per-branch state + disk, PostgreSQL version, health flags.
- Ensure handlers are idempotent where the operation is (switch, ping).
- All handlers must not leak secrets in results/logs (issue 22).

## Acceptance Criteria
- Each CLI command produces correct behavior end-to-end through the daemon.
- Error codes are stable and documented; the CLI maps them to actionable text.
- `status`/`list` `--json` output matches the documented result schemas.

## Validation
- Integration on macOS CI exercising each method through the real socket.
- Golden-file tests for `--json` result schemas.

## Dependencies
- 12, and the operations: 06, 09, 11, 15, 18.

## Non-goals
- New behavior beyond wiring; transport (12).

## Design References
- [DESIGN §7.2](../DESIGN.md#72-control-protocol), [§12](../DESIGN.md#12-cli-surface-issue-map)
