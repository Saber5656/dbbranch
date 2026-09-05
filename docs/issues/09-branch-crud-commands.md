# 09 — Branch CRUD: `create`, `list`, `delete`, `reset`

## Title
Implement branch database create/list/delete/reset (no proxy switching)

## Summary
Implement the branch-management operations that create, enumerate, delete, and
reset per-branch database clones, updating state accordingly. These operate on
clones and instances but do not change the active proxy target (that is issue 11).

## Context
Branch lifecycle & state machine: [DESIGN §6.1](../DESIGN.md#61-per-branch-state-machine),
CLI surface: [DESIGN §12](../DESIGN.md#12-cli-surface-issue-map).

## Scope
- In: `create`, `list`, `delete`, `reset` command logic + daemon-side operations
  that combine clone engine (08), instance manager (07), and state (03).
- Out: proxy repoint / activation (issue 11); daemon RPC transport (issue 12/13)
  — logic here is invoked by the daemon.

## Detailed Requirements
- `Create(branch string, from string)`:
  - Validate/encode branch name (issue 04). Reject if already exists.
  - Resolve source: `from` (a branch or `base`); default `base_branch`. If the
    named source branch's clone is missing, fall back to `base` and warn.
  - If source is a running branch, pass `SourceRunning=true` + quiesce/resume
    callbacks to the clone engine (issue 08). Cloning from `base` uses the
    stopped base directly.
  - Transition `absent→cloning→stopped`; write branch record (`createdFrom`).
- `List()`: return all branches with `state`, `createdFrom`, `clonedAt`,
  `lastActiveAt`, disk usage (approx via `du`-like walk or APFS clone-aware size;
  document that CoW-shared blocks make naive size misleading — report logical +
  note). Mark the active branch.
- `Delete(branch string)`:
  - Refuse if `branch` is the active branch (must switch away first).
  - If running, `Stop` (issue 07). Remove `BranchDir` (clone + socket + logs).
    Remove the branch record. `→ absent`.
  - Refuse to delete `base_branch`'s live clone only if it is active; otherwise
    allowed (it re-clones from base on next use). Never delete `base/pgdata`.
- `Reset(branch string)`:
  - Stop the instance if running. Remove the clone. Re-clone from `base`. Keep
    the branch record (state `stopped`). If the branch was active, it must be
    re-started/re-activated by a subsequent switch (document; or auto-restart if
    active — choose: auto re-clone + restart + keep active, and repoint proxy via
    issue 11 API).

## Acceptance Criteria
- `create feature --from base` produces a `stopped` clone; `list` shows it.
- `create b2 --from feature` clones from feature (quiescing it briefly if running).
- `delete` on a non-active branch removes all its artifacts and the record;
  deleting the active branch is refused with a clear message.
- `reset` returns a branch's data to base contents.
- `list --json` emits stable machine-readable output including the active marker.

## Validation
- Integration on macOS CI: create/list/delete/reset cycle with data assertions
  (write to a branch, reset, confirm base contents restored; confirm base and
  sibling branches unaffected).

## Dependencies
- 03, 04, 07, 08. (Activation coupling with 11 for reset-of-active.)

## Non-goals
- Switching the proxy target (issue 11).

## Design References
- [DESIGN §6.1](../DESIGN.md#61-per-branch-state-machine), [§12](../DESIGN.md#12-cli-surface-issue-map)
