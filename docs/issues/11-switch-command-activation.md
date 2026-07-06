# 11 — `dbbranch switch`: activation orchestration

## Title
Implement `dbbranch switch` to clone/start as needed and repoint the proxy

## Summary
Implement the switch operation that makes a branch the active proxy target:
ensure its clone exists, ensure its instance is running, set it active, and
repoint the proxy — the central orchestration tying together clone, instance,
proxy, and state.

## Context
[DESIGN §6.2 switch sequence](../DESIGN.md#62-project-level-switch-sequence),
[ADR-002](../decisions/ADR-002-fixed-port-l4-proxy.md).

## Scope
- In: switch orchestration (daemon-side) + `switch` CLI command; `--branch`
  resolution from git; `--if-managed`, `--quiet`, `--from` flags.
- Out: transport (issue 12/13); proxy internals (issue 10); clone internals (08).

## Detailed Requirements
- CLI `switch [<branch>] [--from <src>] [--if-managed] [--quiet]`:
  - If `<branch>` omitted, resolve current git branch via
    `git symbolic-ref --short HEAD`. Detached HEAD → error (or, with
    `--if-managed`, a clean exit 0 no-op for the hook, issue 17).
  - `--if-managed`: if the project isn't dbbranch-initialized or the branch is
    intentionally unmanaged, exit 0 silently (hook-safe).
- Daemon `Switch(branch, from string)` sequence (under project lock),
  implementing [DESIGN §6.2](../DESIGN.md#62-project-level-switch-sequence):
  1. Resolve/encode branch. If absent → `Create` (issue 09) from `from`/base.
  2. If `stopped` → `Start` + `WaitReady` (issue 07). On failure → `error` state,
     return with log tail.
  3. `proxy.SetTarget(<branch socket>)` (issue 10); set
     `state.Proxy.ActiveBranch = branch`; update `lastActiveAt`.
  4. Persist state. Return a status payload (active branch, connection string).
  - Idempotent: switching to the already-active running branch returns success
    without side effects.
- Old active instance is left running to drain (issue 10/15 handle stop).

## Acceptance Criteria
- `switch feature` on a fresh branch clones from base, starts it, and makes it
  reachable at the stable port; `status` shows `activeBranch=feature`.
- `switch main` afterward routes new connections back to main's untouched data.
- Switching to a branch whose start fails leaves a clear error + `error` state,
  and the previously active branch keeps serving.
- `switch` with no arg uses the current git branch; detached HEAD errors (or
  no-ops under `--if-managed`).

## Validation
- E2E on macOS CI: init → switch A → write marker → switch B → confirm B is clean
  → switch A → confirm marker present. Assert connection string never changed.

## Dependencies
- 03, 04, 07, 08, 09, 10; consumed via daemon (12/13).

## Non-goals
- Idle-stopping the drained instance (issue 15); the hook (issue 17).

## Design References
- [DESIGN §6.2](../DESIGN.md#62-project-level-switch-sequence)
