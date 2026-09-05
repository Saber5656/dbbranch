# Research: Neon's branching model vs. dbbranch's local analog

Date: 2026-07-06. Basis for the "local Neon" framing and
[ADR-001](../decisions/ADR-001-cow-clone-per-branch-instance.md).

## How Neon branches

- Neon **separates storage and compute** and replaces PostgreSQL's storage layer
  with a log-structured, page-level copy-on-write engine backed by object storage.
- The engine **never overwrites pages in place**; it appends new page versions
  and tracks all changes via the PostgreSQL WAL.
- A **branch is a metadata pointer** to a point in the parent's WAL history.
  Creating a branch moves pointers/metadata rather than copying data pages.
- Branching is therefore **O(1)** regardless of database size (a 1 TB database
  branches in under a second); the branch initially references the parent's pages
  and only diverges for newly written pages.
- The Pageserver caches hot pages on SSD; cold data lives in S3.

## dbbranch's local analog and how it differs

| Aspect | Neon | dbbranch (v1) |
|---|---|---|
| CoW granularity | Page-level, in a custom storage engine | Whole-cluster, at the APFS filesystem level |
| Branch = | WAL-history pointer (metadata) | An APFS clone of the entire `PGDATA` |
| Instant regardless of size | Yes | Yes (APFS clone is O(1)) |
| Shared history / time-travel | Yes (log-structured) | No (v1) |
| Compute | Serverless, scale-to-zero, remote | One local PostgreSQL process per branch |
| Storage | Bottomless (S3) | Local APFS volume |
| Scope | Cloud, multi-tenant | Local, single-user, loopback |

Takeaway: dbbranch reproduces the **developer-visible property** that matters
locally — instant, isolated, per-branch databases — using commodity APFS CoW
instead of a bespoke storage engine. It intentionally does **not** attempt
page-level sharing, time-travel, or scale-to-zero in v1
([DESIGN §14](../DESIGN.md#14-v2--deferred-ideas-not-in-v1)).

## Sources

- [neondatabase/neon — GitHub](https://github.com/neondatabase/neon)
- [Neon Storage: Bottomless, Branchable](https://neon.com/storage)
- [Instantly Copy TB-Size Datasets: The Magic of Copy-on-Write — Neon](https://neon.com/blog/instantly-copy-tb-size-datasets-the-magic-of-copy-on-write)
