# ADR-001: APFS copy-on-write clone of PGDATA + one PostgreSQL instance per branch

- Status: Accepted
- Date: 2026-07-06
- Deciders: Project owner (human), Fable (design)

## Context

dbbranch must make per-git-branch PostgreSQL databases that are created
**instantly regardless of data size** ("local Neon"). Candidate mechanisms:

1. **CoW filesystem clone of `PGDATA`** + a PostgreSQL instance per branch.
2. **`CREATE DATABASE ... TEMPLATE`** inside one instance.
3. Dump/restore (`pg_dump`/`pg_restore`).
4. Running actual Neon (pageserver/compute) locally.

Neon achieves O(1) branching by making a branch a metadata pointer into a
log-structured, page-level CoW storage engine (verified: [research](../research/neon-branching-model.md)).
We cannot replicate page-level CoW without reimplementing Neon's storage layer.
The closest local analog with commodity tools is **filesystem-level** CoW.

## Decision

Use **APFS copy-on-write clones of the entire `PGDATA` directory**, one clone
per branch, each served by its **own PostgreSQL instance**.

## Rationale

- Cloning is O(1) in logical data size on APFS; clones diverge lazily block by
  block. This delivers the "instant regardless of size" promise.
- Whole-cluster clones give true isolation: every branch has independent WAL,
  catalogs, settings, and extensions — destructive migrations cannot leak.
- Template databases (option 2) physically copy at O(data) and require no
  connections to the template; they are neither instant for large data nor CoW.
  Rejected as the core mechanism (may appear as a v2 fast-path for tiny DBs).
- Dump/restore is O(data) and slow. Rejected.
- Embedding Neon is heavyweight and complex for a local dev tool. Deferred (v2+).

## Consequences

- Requires per-branch process + port/socket management and a proxy for a stable
  connection string ([ADR-002](ADR-002-fixed-port-l4-proxy.md)).
- Coarser granularity than Neon: no shared page history, no time-travel in v1;
  branches share physical blocks only until first write.
- Ties v1 to APFS and macOS ([ADR-004](ADR-004-macos-apfs-only-v1.md)).
- Clone consistency must be managed ([ADR-005](ADR-005-clone-consistency-strategy.md)).
- Directory cloning must use `copyfile(3)` `COPYFILE_CLONE_FORCE`, not raw
  `clonefile(2)` (Apple guidance; [research](../research/apfs-clonefile-and-consistency.md)).
