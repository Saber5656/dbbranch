# 02 — Committed config parser & validation (`.dbbranch/config.toml`)

## Title
Parse and strictly validate the untrusted committed `.dbbranch/config.toml`

## Summary
Implement `internal/config` loading of the repo-committed `.dbbranch/config.toml`
with strict, security-first validation. This file is **untrusted** (ships in
cloned repos), so parsing must reject anything unexpected and must never allow
binary/command execution or path escape.

## Context
See [DESIGN §5.4](../DESIGN.md#54-configtoml-committed-untrusted-schema) and the
threat model ([DESIGN §11](../DESIGN.md#11-security-model)). The committed config
holds only shareable, declarative settings.

## Scope
- In: schema struct, TOML decode, field validation, typed errors.
- Out: local/global trusted config (issue 05 uses it; struct may live here but
  loading of the trusted file is separate), applying config to actions.

## Detailed Requirements
- Use a TOML library with **strict/unknown-key detection** (e.g.
  `github.com/BurntSushi/toml` `Decode` + `MetaData.Undecoded()` check, or
  `DisallowUnknownFields` equivalent). Unknown keys → error listing them.
- Struct `RepoConfig` mirroring [DESIGN §5.4](../DESIGN.md#54-configtoml-committed-untrusted-schema):
  `Version int`, `PostgresVersion string`, `Database string`,
  `BaseBranch string`, `ProxyPort int`, `IdleTimeout string`, `MaxInstances int`,
  `Seed struct{ Strategy string; SQLPath string }`.
- Validation rules (all return typed `ConfigError` with field + reason):
  - `Version` required and must equal `1`.
  - `PostgresVersion` required, matches `^[0-9]{1,2}$`.
  - `Database` required, matches `^[A-Za-z_][A-Za-z0-9_]{0,62}$`.
  - `BaseBranch` optional; if set, a valid git ref name (no spaces, control chars,
    `..`, leading/trailing `/`, no `~^:?*[\`).
  - `ProxyPort` optional; if set, `1024 <= port <= 65535`.
  - `IdleTimeout` optional; parseable Go duration or `"0"`.
  - `MaxInstances` optional; `1..64`.
  - `Seed.Strategy` in `{"empty","sql"}` (default `"empty"`).
  - If `Strategy=="sql"`: `SQLPath` required; must be repo-relative (no leading
    `/`), contain no `..` segment after `filepath.Clean`, and resolve inside the
    repo root. Provide `ResolveSeedPath(repoRoot)` returning the safe absolute
    path or an error.
  - **No field may be an absolute path or reference a binary/command.**
- Provide `Load(repoRoot string) (*RepoConfig, error)`; missing file is a
  distinct `ErrConfigNotFound` (callers decide if that's fatal).
- Provide `Defaults()` and a merged-effective view (defaults applied).

## Acceptance Criteria
- Valid config decodes into `RepoConfig` with defaults applied.
- Every invalid case above yields a typed error naming the field.
- Unknown keys are rejected.
- `ResolveSeedPath` rejects `../x`, `/etc/passwd`, and symlink escapes; accepts
  `.dbbranch/seed.sql`.

## Validation
- Table-driven unit tests covering each rule (valid + invalid).
- Include a fuzz seed corpus for `Load` (feeds issue 21).

## Dependencies
- 01 (scaffolding).

## Non-goals
- Executing seed SQL (issue 06/18); loading trusted local config (issue 05).

## Design References
- [DESIGN §5.4](../DESIGN.md#54-configtoml-committed-untrusted-schema)
- [DESIGN §11.2](../DESIGN.md#112-abuse-cases--mitigations)
- [Issue 21 — untrusted config hardening](21-security-untrusted-repo-config.md)
