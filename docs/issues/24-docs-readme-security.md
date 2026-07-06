# 24 — Docs: README, SECURITY.md, threat model, CONTRIBUTING

## Title
Write user and contributor documentation, including the security model

## Summary
Author the public-facing documentation: a quickstart README, `SECURITY.md` with
the threat model and reporting process, `CONTRIBUTING.md`, and command usage
reference — aligned with `docs/DESIGN.md`.

## Context
This is an OSS project intended for public release; docs are part of v1
([DESIGN §11](../DESIGN.md#11-security-model)).

## Scope
- In: `README.md` (expand from the current one line), `SECURITY.md`,
  `CONTRIBUTING.md`, `docs/USAGE.md` (command reference), `CODE_OF_CONDUCT.md`.
- Out: the design docs themselves (already in `docs/`).

## Detailed Requirements
- `README.md`: what/why (local Neon), macOS+PostgreSQL requirements, install
  (Homebrew, issue 23), quickstart (the [DESIGN §1.1](../DESIGN.md#11-primary-user-story)
  flow), the stable connection-string UX, limitations (macOS/APFS/PostgreSQL
  only), and a link to `docs/DESIGN.md`.
- `SECURITY.md`: trust boundaries and abuse cases (summarize
  [DESIGN §11](../DESIGN.md#11-security-model) and issues 20–22), the loopback-only
  guarantee, the untrusted-repo-config guarantees (issue 21), the
  passwordless-trust rationale, supported versions, and a private vulnerability
  reporting channel (GitHub Security Advisories).
- `CONTRIBUTING.md`: build/test (`make ci`), the docs-are-source-of-truth rule
  (docs/ over Issues/PRs), branch/PR conventions, that design changes require an
  ADR.
- `docs/USAGE.md`: every command (issue map, [DESIGN §12](../DESIGN.md#12-cli-surface-issue-map))
  with flags, examples, and exit codes.
- Keep docs consistent with `DESIGN.md`; if they diverge, `DESIGN.md` wins and
  must be updated first.

## Acceptance Criteria
- README enables a new macOS user to install, `init`, and get a working
  `DATABASE_URL` from the quickstart alone.
- `SECURITY.md` documents the threat model, the loopback + untrusted-config
  guarantees, and a working private reporting path.
- `USAGE.md` covers every v1 command and flag.
- Markdown links resolve; no references to unimplemented behavior.

## Validation
- Doc lint (markdownlint) + link check in CI.
- Manual walkthrough of the README quickstart on a clean macOS environment.

## Dependencies
- Content depends on the shipped command surface (06/09/11/16/17) and security
  issues (20–22); can be drafted in parallel and finalized before release.

## Non-goals
- API/library docs (dbbranch is a CLI); website.

## Design References
- [DESIGN §11](../DESIGN.md#11-security-model), [§1.1](../DESIGN.md#11-primary-user-story), [§12](../DESIGN.md#12-cli-surface-issue-map)
