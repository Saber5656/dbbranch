# 04 — Project identity, home directory layout & branch-name encoding

## Title
Resolve git project identity, the dbbranch home layout, and branch↔dir encoding

## Summary
Implement `internal/paths` (or `internal/project`): git repo-root detection,
project-id derivation, the `~/.dbbranch/...` directory layout with correct
permissions, and the bijective branch-name↔directory-name encoding.

## Context
Every runtime artifact is keyed by project and branch. See
[DESIGN §5.1–5.3](../DESIGN.md#5-on-disk-layout--data-model).

## Scope
- In: path builders, directory creation with modes, git-root detection,
  project-id hashing, branch-name encoding/decoding, APFS-volume checks helper
  signature (impl in issue 08 can reuse).
- Out: cloning, instance mgmt.

## Detailed Requirements
- `DetectRepoRoot(startDir string) (string, error)`: run
  `git rev-parse --show-toplevel`; `realpath` the result. Not-a-repo → typed
  error. Honor `--project` override (validated to be a git root).
- `ProjectID(canonicalРepoRoot string) string`:
  `hex(sha256(path))[:16]`. Deterministic.
- Home root resolution: `~/.dbbranch` (respect `$DBBRANCH_HOME` override for
  tests). Builders:
  - `Home()`, `ProjectDir(id)`, `BaseDir(id)`, `BranchDir(id, dirName)`,
    `PgdataDir(...)`, `SocketDir(id, dirName)`, `LogsDir(id)`,
    `StatePath(id)`, `ProjectJSONPath(id)`, `DaemonSock()`, `DaemonPID()`.
- `EnsureDirs(...)`: create with `0700` (dirs) / `0600` (files); never widen
  perms on existing dirs; refuse if a component is a symlink dbbranch did not
  create (record ownership marker or check `O_NOFOLLOW` on create).
- Branch encoding ([DESIGN §5.2](../DESIGN.md#52-branch-name--directory-name-mapping)):
  - `EncodeBranch(name) (dirName string, err error)`: `/`→`__`, then
    percent-encode chars outside `[A-Za-z0-9._-]`. Reject encoded length > 200,
    reserved names (`base, run, logs, ., ..`), empty.
  - `DecodeBranch(dirName) string` — but callers should read the stored
    `name` from state; decoding is a fallback and must be exact-inverse.
  - Property test: `Decode(Encode(x)) == x` for a fuzzed corpus of branch names.
- APFS check helper signature: `IsAPFSSameVolume(a, b string) (bool, error)`
  (implementation may be stubbed here and completed in issue 08).

## Acceptance Criteria
- `DetectRepoRoot` returns the canonical toplevel; errors outside a repo.
- `ProjectID` is stable and 16 hex chars.
- Directory builders produce exactly the layout in DESIGN §5.1.
- `EnsureDirs` yields `0700`/`0600` perms; symlink components are refused.
- Encoding is bijective on the fuzz corpus; reserved/oversized names rejected.

## Validation
- Unit + property tests for encoding.
- Perms test: created dirs are `0700`, files `0600`.
- Test with `feature/x`, `release/1.2`, unicode, and `HEAD~`-like names.

## Dependencies
- 01.

## Non-goals
- Clone/volume implementation details (issue 08).

## Design References
- [DESIGN §5.1–5.3](../DESIGN.md#5-on-disk-layout--data-model)
