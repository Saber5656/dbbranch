# 01 — Repo scaffolding & CLI skeleton

## Title
Scaffold the Go module, repository layout, tooling, and `dbbranch` CLI skeleton

## Summary
Create the Go module, directory structure, build/lint/test tooling, macOS CI, and
a Cobra-based `dbbranch` CLI that exposes `--version`/`version` and registers
(stubbed) subcommands. This is the foundation every other issue builds on.

## Context
dbbranch is a Go CLI + daemon for macOS (see [DESIGN §4](../DESIGN.md#4-architecture-overview)).
Nothing exists yet beyond `README.md`. We need a clean, conventional Go layout so
later issues can add packages without churn.

## Scope
- In: module init, directory layout, Makefile/task runner, linting, macOS CI,
  CLI entrypoint with global flags and command registration, version wiring.
- Out: any real command behavior (stubs return "not implemented"); the daemon.

## Detailed Requirements
- `go.mod` with module path `github.com/<owner>/dbbranch` (owner filled at repo
  creation; use `github.com/dbbranch/dbbranch` as placeholder, documented in
  README). Go version: latest stable 1.x pinned in `go.mod`.
- Directory layout:
  ```
  cmd/dbbranch/main.go        # CLI entrypoint
  cmd/dbbranchd/main.go       # daemon entrypoint (stub prints "not implemented")
  internal/cli/               # cobra commands
  internal/version/           # version vars set via -ldflags
  internal/config/            # (created here as empty package doc)
  internal/state/
  internal/pg/
  internal/pgfind/
  internal/clone/
  internal/proxy/
  internal/daemon/
  Makefile
  .golangci.yml
  .github/workflows/ci.yml
  ```
- CLI (`internal/cli`) built on `spf13/cobra`. Root command `dbbranch` with
  global persistent flags: `--json`, `--quiet`, `--verbose`, `--project <path>`.
- Register subcommands as stubs (each returns exit code 3 + "not implemented":
  `init, status, url, connection, switch, create, list, delete, reset, gc,
  base, hook, daemon, doctor, version`.
- `internal/version`: `Version`, `Commit`, `BuildDate` string vars; `-ldflags`
  in Makefile set them. `dbbranch version` and `dbbranch --version` print
  `dbbranch <version> (<commit>, <date>, <goversion>, <os>/<arch>)`.
- `Makefile` targets: `build`, `test`, `lint`, `fmt`, `vet`, `ci` (runs
  fmt-check+vet+lint+test+build).
- `.golangci.yml`: enable `govet, staticcheck, errcheck, ineffassign,
  gofmt, goimports, revive`.
- `.github/workflows/ci.yml`: run on `macos-latest`, steps: checkout, setup-go,
  `make ci`. Trigger on push + PR.
- Add `LICENSE` (MIT placeholder), `.gitignore` (Go + `.dbbranch/` local).

## Acceptance Criteria
- `make ci` passes locally on macOS.
- `go build ./...` produces `dbbranch` and `dbbranchd` binaries.
- `dbbranch --version` and `dbbranch version` print populated version info.
- Every listed subcommand exists and exits with code 3 + "not implemented".
- CI workflow is green on macos-latest.

## Validation
- Run `make ci`.
- Run `./dbbranch version`, `./dbbranch switch` (expect exit 3), `./dbbranch --help`.

## Dependencies
- None (root issue).

## Non-goals
- Real command logic, daemon logic, any PostgreSQL interaction.

## Design References
- [DESIGN §4 Architecture](../DESIGN.md#4-architecture-overview)
- [DESIGN §12 CLI surface](../DESIGN.md#12-cli-surface-issue-map)
