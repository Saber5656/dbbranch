# ADR-005: Clone consistency via source quiesce in v1; low-level backup API deferred

- Status: Accepted
- Date: 2026-07-06
- Deciders: Fable (design), pending human ratification

## Context

Cloning `PGDATA` while PostgreSQL is writing to it produces a copy equivalent to
a crash. PostgreSQL can recover from a **crash-consistent atomic** copy, but a
`copyfile` walk of a live directory is not atomic across the tree and may capture
a torn state. Options for consistency:

- **A. Quiesce the source** (stop the instance) before cloning, then restart it.
- **B. Low-level online backup**: `pg_backup_start(label, fast)` → clone →
  `pg_backup_stop()`, honoring the returned backup-label/WAL requirements
  (function names are the PostgreSQL ≥15 spelling; verified in
  [research](../research/apfs-clonefile-and-consistency.md)).
- **C. `CHECKPOINT` then clone live**, relying on crash recovery (still risks a
  torn, non-atomic copy).

## Decision

- The **base** cluster is kept **quiesced** whenever it is used as a clone source
  (its instance is stopped before it serves as a source). Clone-from-base — the
  common path — is therefore trivially consistent.
- **Clone-from-a-running-branch** (`--from <live-branch>`) uses **option A**
  (quiesce-then-clone): fast-stop the source, clone, restart the source.
- **Option B is deferred to v2** as a zero-downtime optimization.
- After cloning, always delete the clone's `postmaster.pid` before first start
  and allow crash recovery to complete.

## Rationale

- Correctness first: option A is simple and provably consistent; the common
  clone-from-base path has no downtime because base is already stopped.
- Option B adds WAL-window handling and label/tablespace-map placement
  complexity that is not justified for a local single-machine tool in v1.
- Option C alone is unsafe (non-atomic tree copy) and is rejected.

## Consequences

- Cloning from a *live* branch briefly interrupts only that source instance.
- v2 can add option B behind a flag for zero-downtime forks of large live
  branches ([DESIGN §14](../DESIGN.md#14-v2--deferred-ideas-not-in-v1)).
- The clone engine (issue 08) must expose a "quiesce source" hook the daemon
  drives before cloning a running instance.
