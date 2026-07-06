# Research: APFS clone semantics & PostgreSQL filesystem-copy consistency

Date: 2026-07-06. Basis for [ADR-001](../decisions/ADR-001-cow-clone-per-branch-instance.md),
[ADR-004](../decisions/ADR-004-macos-apfs-only-v1.md),
[ADR-005](../decisions/ADR-005-clone-consistency-strategy.md), and
[DESIGN §9](../DESIGN.md#9-clone-engine-internalclone-issue-08).

## APFS copy-on-write clones

- A clone shares the source's data blocks but has its own attributes/xattrs/ACLs.
  Subsequent writes to either side are **private** (copy-on-write); the two
  diverge lazily, only for modified blocks.
- Cloning is **O(1)** in logical size — this is what gives "instant regardless of
  data size".
- **Directory hierarchies**: Apple's `clonefile(2)` man page states that using
  `clonefile(2)` to clone directory hierarchies is **strongly discouraged**, and
  `copyfile(3)` should be used for copying directories. dbbranch therefore uses
  `copyfile(3)` with `COPYFILE_CLONE_FORCE` (recursive), which performs a CoW
  clone where possible and **fails** (rather than silently deep-copying) when a
  true clone is impossible.
- **Same-volume requirement**: clones require source and destination on the same
  APFS volume. Cross-device attempts fail (`cp -c` reports
  `clonefile failed: Cross-device link`). dbbranch preflights `st_dev`/`statfs`.
- The `cp -c` flag is the command-line equivalent (forces CoW clone on APFS).

Implication for dbbranch: clone the whole `PGDATA` tree with a recursive
`copyfile(3)` CoW clone, on one APFS volume, and treat a clone-force failure as a
hard, explained error (non-APFS or cross-volume).

## PostgreSQL consistency of a filesystem copy

- A copy of a **running** cluster's data directory is only safely recoverable if
  it is **crash-consistent and atomic**. A non-atomic tree walk can capture a
  torn state.
- The supported online method is the **low-level backup API**. In PostgreSQL
  **15+** the functions were **renamed**:
  - `pg_start_backup()` → `pg_backup_start(label, fast)`
  - `pg_stop_backup()` → `pg_backup_stop()`
  - Exclusive backup mode was **removed** in 15; `pg_is_in_backup()` and
    `pg_backup_start_time()` were removed with it.
  - `fast=true` forces an immediate checkpoint (faster, I/O spike).
  - Procedure: `pg_backup_start` → copy the data directory by a reliable method →
    `pg_backup_stop` (returns the backup label + tablespace map, which must be
    placed in the copy), plus the WAL generated during the window must be
    available for replay.
- The simplest provably-consistent local method is to **stop** the source
  cluster (clean shutdown checkpoints), then clone the quiesced directory. A
  clone of a cleanly-stopped cluster needs no recovery.

Implication for dbbranch (v1): keep the base quiesced and clone it directly;
quiesce-then-clone when forking a live branch; defer the low-level backup API
(zero-downtime) to v2. Always remove `postmaster.pid` from a clone before start.

## Sources

- [clonefile(2) / APFS clone semantics — Eclectic Light](https://eclecticlight.co/2020/04/14/copy-move-and-clone-files-in-apfs-a-primer/)
- [Copy-on-write on APFS — Wade Tregaskis](https://wadetregaskis.com/copy-on-write-on-apfs/)
- [APFS clone using `cp -c` — Apple Community](https://discussions.apple.com/thread/250375970)
- [Exclusive Backup Mode Removed in Postgres 15 — EDB](https://www.enterprisedb.com/blog/exclusive-backup-mode-finally-removed-postgres-15)
- [PostgreSQL 15: low level backup function changes — fluca1978](https://fluca1978.github.io/2022/07/13/PostgreSQL15BackupFunctions.html)
- [pg_backup_start() — pgPedia](https://pgpedia.info/p/pg_backup_start.html)
- [PostgreSQL 15 release notes](https://www.postgresql.org/docs/15/release-15.html)
