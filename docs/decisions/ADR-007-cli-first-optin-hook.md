# ADR-007: CLI-first activation; Git hook is opt-in and non-blocking

- Status: Accepted
- Date: 2026-07-06
- Deciders: Project owner (human), Fable (design)

## Context

How does switching git branches drive switching database branches? Options:

- **A. Manual CLI first**, with an opt-in `post-checkout` hook for automation.
- **B. Fully automatic via git hook** installed by default.
- **C. A daemon that watches `.git/HEAD`** and switches with no CLI/hook.

Human decision (recorded): **A.**

## Decision

The canonical activation is the explicit CLI (`dbbranch switch`). A
`post-checkout` Git hook is available via `dbbranch hook install` (opt-in only).
The hook is minimal, time-bounded, and **must never block or fail `git`**.

## Rationale

- Predictable, auditable behavior; no surprise database switches from a `git`
  operation the user did not associate with dbbranch.
- Opt-in installation avoids writing into `.git/hooks` without explicit consent
  (a supply-chain / least-surprise concern).
- HEAD-watching (option C) adds a privileged always-watching process with more
  failure and security surface; deferred to v2.

## Consequences

- The hook calls `dbbranch switch --if-managed --quiet` and always `exit 0`
  ([issue 17](../issues/17-git-post-checkout-hook.md)).
- Detached HEAD and non-branch checkouts make the hook a no-op.
- Fully automatic HEAD-watching remains a documented v2 option
  ([DESIGN §14](../DESIGN.md#14-v2--deferred-ideas-not-in-v1)).
