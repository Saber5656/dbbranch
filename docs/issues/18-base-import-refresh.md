# 18 — `dbbranch base import` and base refresh

## Title
Implement loading/refreshing the base cluster from a SQL dump

## Summary
Implement `dbbranch base import <dump.sql>` to (re)initialize the base database
contents from a SQL dump, which becomes the clone source for future branches.

## Context
The base is the template all branches clone from
([DESIGN §5.1](../DESIGN.md#51-directory-layout),
[DESIGN §12](../DESIGN.md#12-cli-surface-issue-map)).

## Scope
- In: `base import` command + operation; safe reset of base contents; validation.
- Out: initial `init` (issue 06, which may call a shared helper); adopting a live
  external cluster (v2).

## Detailed Requirements
- `dbbranch base import <dump.sql> [--yes]`:
  - Refuse if any branch other than the base is active/running unless the user
    confirms, because existing branch clones will diverge from the new base
    (existing clones are unaffected physically; only future clones use the new
    base). Print a clear explanation.
  - Validate `<dump.sql>` exists and is readable. It is a **user-supplied local
    path** on the CLI (trusted as a user action), distinct from the untrusted
    repo `seed.sql`. Still run `psql -v ON_ERROR_STOP=1`.
  - Procedure: ensure base is stopped → start base on private socket → drop &
    recreate `<database>` (or `--no-recreate` to load into existing) → load dump →
    stop base (quiesced).
  - On any failure, restore the base to a usable state (do not leave a
    half-dropped database): perform load into a temporary database first, then
    atomically swap by rename, or wrap in a transaction where feasible; document
    the chosen safe strategy.
- Optional `dbbranch base refresh`: re-run the committed `seed.sql` strategy
  (issue 02) into a fresh base (same safety rules; untrusted seed still executed
  only via this explicit command, never on switch).

## Acceptance Criteria
- `base import dump.sql` results in a base whose `<database>` matches the dump;
  new branches created afterward contain the imported data.
- A failing dump leaves the previous base intact and usable.
- Importing while other branches run is gated by confirmation with a clear note.

## Validation
- Integration on macOS CI: import a dump, create a branch, assert the branch has
  the dump's data; import a deliberately broken dump and assert base is unchanged.

## Dependencies
- 03, 06, 07, 09.

## Non-goals
- Live external cluster adoption; cross-version import (v2).

## Design References
- [DESIGN §5.1](../DESIGN.md#51-directory-layout), [§9.2](../DESIGN.md#92-consistency-adr-005)
