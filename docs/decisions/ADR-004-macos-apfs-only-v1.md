# ADR-004: macOS + APFS only for v1

- Status: Accepted
- Date: 2026-07-06
- Deciders: Project owner (human), Fable (design)

## Context

The core mechanism is filesystem CoW cloning
([ADR-001](ADR-001-cow-clone-per-branch-instance.md)). CoW is filesystem-specific:
APFS on macOS, ZFS/Btrfs/overlayfs on Linux, ReFS/dev-drive on Windows. Human
decision (recorded): **macOS only for v1.**

## Decision

v1 targets **macOS** (Apple Silicon and Intel) on **APFS** volumes only. CI runs
on macOS runners. Non-APFS volumes and non-macOS platforms are refused with
actionable errors.

## Rationale

- The owner's environment is macOS; APFS `copyfile(3)`/`clonefile(2)` CoW is
  standard and reliable.
- Constraining to one filesystem removes a large matrix of CoW backends and
  environment detection from v1, maximizing v1 quality and issue executability.
- Cross-platform requires a storage abstraction that is better designed once the
  single-backend design is proven.

## Consequences

- `dbbranch doctor`/`init` must verify the dbbranch home and base live on an
  APFS volume and that source/dest of any clone share one volume
  ([DESIGN §9.1](../DESIGN.md#91-primitive)).
- No Linux/Windows CI or packaging in v1.
- Linux (ZFS/Btrfs) and a storage abstraction are explicit v2 items
  ([DESIGN §14](../DESIGN.md#14-v2--deferred-ideas-not-in-v1)).
