# 08 — APFS copy-on-write clone engine

## Title
Implement the APFS CoW clone of a `PGDATA` directory with consistency handling

## Summary
Implement `internal/clone`: recursively CoW-clone a `PGDATA` directory on APFS
using `copyfile(3)` `COPYFILE_CLONE_FORCE`, with same-volume/APFS preflight, a
"quiesce source" contract for live sources, and post-clone hygiene.

## Context
This is the core "instant branch" mechanism
([ADR-001](../decisions/ADR-001-cow-clone-per-branch-instance.md),
[ADR-005](../decisions/ADR-005-clone-consistency-strategy.md),
[DESIGN §9](../DESIGN.md#9-clone-engine-internalclone-issue-08),
[research](../research/apfs-clonefile-and-consistency.md)).

## Scope
- In: clone primitive binding, APFS/volume checks, clone orchestration with a
  quiesce callback, post-clone cleanup, failure handling.
- Out: deciding when to clone / from where (issues 09/11); stopping the source
  process (issue 07 provides `Stop`; this issue only calls a provided callback).

## Detailed Requirements
- Clone primitive `cloneTree(src, dst string) error`:
  use `copyfile(3)` with flags `COPYFILE_CLONE_FORCE | COPYFILE_RECURSIVE`
  (via cgo binding to `<copyfile.h>` or `golang.org/x/sys/unix` if available).
  **Do not** use raw `clonefile(2)` for directory trees (Apple discourages it).
  `COPYFILE_CLONE_FORCE` must **fail** rather than deep-copy when a clone is
  impossible; treat that failure as `ErrCloneUnsupported`.
- `IsAPFSSameVolume(src, dst string) (bool, error)`: compare `Stat_t.Dev`;
  confirm the filesystem type is `apfs` via `statfs` (`f_fstypename`). Non-APFS or
  cross-volume → typed error with remediation.
- `Clone(opts CloneOptions) error` where
  `CloneOptions{ SrcPgdata, DstPgdata string; SourceRunning bool;
  Quiesce func() error; Resume func() error }`:
  1. Preflight `IsAPFSSameVolume(srcParent, dstParent)`.
  2. If `SourceRunning`: call `Quiesce()` (daemon passes `pg.Stop`); ensure
     `Resume()` runs via defer even on error.
  3. `cloneTree(SrcPgdata, DstPgdata)`.
  4. Post-clone hygiene: remove `DstPgdata/postmaster.pid`; remove any stale
     socket files under the cloned tree; leave WAL intact (crash recovery on
     first start is expected and allowed).
  5. On any failure, `os.RemoveAll(DstPgdata)` (no partial clones left).
- Concurrency: caller holds the project lock; this package assumes single-writer.

## Acceptance Criteria
- Cloning a stopped base `PGDATA` on APFS creates a startable clone that boots
  clean (issue 07 `Start`+`WaitReady` succeed against the clone).
- Cloned pages are CoW (verify near-instant clone time on a multi-GB test dir;
  divergence only on write).
- Non-APFS/cross-volume source or dest yields a typed, actionable error and
  creates nothing.
- A failed clone leaves no `DstPgdata`.
- `postmaster.pid` is absent in the clone.

## Validation
- Integration on macOS CI: clone a seeded base, boot the clone, verify data
  matches and that writing to the clone does not affect the base.
- Timing assertion: clone of a large (e.g. ≥1 GB) dir completes in well under a
  second (CoW), demonstrating O(1) behavior.
- Negative test: clone onto a non-APFS/RAM disk of a different volume → error.

## Dependencies
- 01, 04, 07 (Stop/Start for the boot check and quiesce callback).

## Non-goals
- Zero-downtime live clone via low-level backup API (v2, [ADR-005](../decisions/ADR-005-clone-consistency-strategy.md)).

## Design References
- [DESIGN §9](../DESIGN.md#9-clone-engine-internalclone-issue-08)
- [research/apfs-clonefile-and-consistency.md](../research/apfs-clonefile-and-consistency.md)
