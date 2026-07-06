# 23 — Packaging: Homebrew tap, release build & provenance

## Title
Build, version-stamp, and distribute dbbranch via a Homebrew tap

## Summary
Produce reproducible macOS release builds (arm64 + amd64) with embedded version
provenance, checksums, and a Homebrew formula/tap for `brew install`.

## Context
Distribution for a macOS OSS tool
([DESIGN §2.1](../DESIGN.md#21-v1-scope-in), [§12](../DESIGN.md#12-cli-surface-issue-map)).

## Scope
- In: release build script/workflow, version ldflags, checksums, Homebrew
  formula, install docs.
- Out: auto-update mechanism (Homebrew handles updates).

## Detailed Requirements
- Release build: `make release` (or GoReleaser) producing signed-where-possible
  archives for `darwin/arm64` and `darwin/amd64`, containing `dbbranch` and
  `dbbranchd`, with `-ldflags` setting `internal/version` vars (issue 01) from
  the git tag/commit/date.
- Generate `SHA256SUMS` for all artifacts.
- GitHub Actions release workflow triggered on tag `v*`: build, checksum, create
  a GitHub Release with artifacts. (Signing/notarization documented as a
  follow-up if Apple Developer ID is available.)
- Homebrew formula (`Formula/dbbranch.rb`) in a tap
  (`homebrew-dbbranch`, documented) that:
  - installs both binaries, declares `depends_on macos`, and documents that a
    user-provided `postgresql@NN` is required at runtime (not a hard dep, since
    the user chooses the version — see ADR-003; may `depends_on "postgresql@16"`
    as a recommended default, documented).
  - includes a `test do` invoking `dbbranch version`.
- Docs: install instructions in README (issue 24) referencing the tap.

## Acceptance Criteria
- Tagging `vX.Y.Z` yields a GitHub Release with arm64+amd64 archives and
  `SHA256SUMS`.
- `dbbranch version` in a released binary reports the correct tag/commit/date.
- `brew install <tap>/dbbranch` installs and `dbbranch version` runs (validated
  in a formula test or documented manual check).

## Validation
- Dry-run the release workflow on a pre-release tag; verify artifacts + checksums.
- `brew audit`/`brew test` the formula where feasible.

## Dependencies
- 01 (version wiring); ideally after core commands exist, but packaging can be
  built in parallel and finalized last.

## Non-goals
- Notarization automation (documented follow-up); non-macOS packaging (v2).

## Design References
- [DESIGN §2.1](../DESIGN.md#21-v1-scope-in), [ADR-003](../decisions/ADR-003-user-provided-postgres.md)
