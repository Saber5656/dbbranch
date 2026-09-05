# 03 — Local state store (`state.json`) with atomic writes & locking

## Title
Implement the atomic, file-locked per-project state store

## Summary
Implement `internal/state`: the authoritative daemon-owned `state.json` per
project, with a typed schema, atomic writes, advisory file locking, and schema
versioning.

## Context
State is the single source of runtime truth (branches, ports, pids, active
pointer). See [DESIGN §5.6](../DESIGN.md#56-statejson-schema-authoritative-daemon-owned).

## Scope
- In: Go structs, load/save, atomic write, `flock`, schema version gate,
  in-memory mutation helpers.
- Out: deciding directory paths (issue 04), business logic that mutates state
  (issues 09/11/etc.).

## Detailed Requirements
- Structs mirroring [DESIGN §5.6](../DESIGN.md#56-statejson-schema-authoritative-daemon-owned):
  `State{ SchemaVersion, ProjectID, RepoRoot, CreatedAt, Postgres, Proxy,
  BaseBranch, Database, Branches map[string]BranchRecord }`.
  `BranchState` enum: `absent, cloning, stopped, starting, running, stopping,
  error` (string constants).
- `Load(path string) (*State, error)`: missing → `ErrStateNotFound`; unknown
  `SchemaVersion > current` → `ErrStateTooNew` (do not touch).
- `Save(path string, s *State) error`: write to `path+".tmp"`, `f.Sync()`,
  `os.Rename` to `path`, with file mode `0600`.
- `Lock(dir string) (release func(), err error)`: `flock` (LOCK_EX) on
  `state.lock` in the project dir; used by the daemon around mutations.
- Mutation helpers (pure, operate on `*State`): `UpsertBranch`, `RemoveBranch`,
  `SetActive`, `SetBranchState`, `SetBranchPID`. Each validates the transition is
  representable (not the full state machine — that lives in the daemon).
- All timestamps RFC3339 UTC. Accept an injected clock for tests.
- `Validate() error` checks internal consistency (active branch exists and is a
  known branch; dirName uniqueness).

## Acceptance Criteria
- Round-trip `Save`→`Load` preserves all fields byte-stably (stable key order via
  struct/JSON marshaling).
- Concurrent `Save` under `Lock` never yields a torn file (temp+rename).
- `ErrStateTooNew` returned for a higher schema version.
- Mutation helpers reject inconsistent inputs (e.g., `SetActive` to unknown
  branch).

## Validation
- Unit tests incl. a concurrency test (goroutines saving under lock; final file
  parses).
- Crash-safety test: interrupt between tmp-write and rename leaves prior file
  intact.

## Dependencies
- 01.

## Non-goals
- Path resolution (issue 04); state machine enforcement (issue 11/daemon).

## Design References
- [DESIGN §5.6](../DESIGN.md#56-statejson-schema-authoritative-daemon-owned)
- [DESIGN §6.1 state machine](../DESIGN.md#61-per-branch-state-machine)
