# 17 — Git `post-checkout` hook (opt-in, non-blocking)

## Title
Implement `dbbranch hook install|uninstall` and the non-blocking post-checkout hook

## Summary
Provide an opt-in `post-checkout` Git hook that calls
`dbbranch switch --if-managed --quiet` on branch checkouts, installed/removed via
`dbbranch hook install|uninstall`, and guaranteed never to block or fail `git`.

## Context
CLI-first with opt-in automation
([ADR-007](../decisions/ADR-007-cli-first-optin-hook.md),
[DESIGN §11.2 abuse case 6](../DESIGN.md#112-abuse-cases--mitigations)).

## Scope
- In: `hook install`/`uninstall` commands; the hook script; the `--if-managed`
  fast path integration in `switch` (issue 11 defines the flag).
- Out: HEAD-watching daemon (v2).

## Detailed Requirements
- `dbbranch hook install`:
  - Locate `.git/hooks` (respect `core.hooksPath`). If a `post-checkout` hook
    exists and is not ours, refuse unless `--force`; on `--force`, append our
    block within clearly delimited markers (`# >>> dbbranch >>>` /
    `# <<< dbbranch <<<`) so uninstall is surgical.
  - Write an executable (`0755`) hook that, only for **branch** checkouts
    (`$3 == 1`), runs, with a timeout and in a way that never blocks git:
    ```sh
    #!/bin/sh
    # >>> dbbranch >>>
    [ "$3" = "1" ] || exit 0
    command -v dbbranch >/dev/null 2>&1 || exit 0
    # fire-and-forget; never block or fail the checkout
    ( dbbranch switch --if-managed --quiet >/dev/null 2>&1 & ) 
    exit 0
    # <<< dbbranch <<<
    ```
  - The hook calls `dbbranch` resolved from PATH; it must not embed an absolute
    path chosen from repo config (issue 21).
- `dbbranch hook uninstall`: remove our block (or the whole file if we own it
  wholesale); leave foreign hook content intact.
- `switch --if-managed` (issue 11): exit 0 silently if not initialized, branch
  unmanaged, or detached HEAD.
- The hook must be idempotent and safe when the daemon is down (switch auto-starts
  it, backgrounded so `git` returns immediately).

## Acceptance Criteria
- `hook install` writes a `0755` `post-checkout` hook (or appends a marked block);
  `uninstall` removes exactly our block.
- After install, `git switch <branch>` triggers a DB switch without delaying or
  failing the checkout, even when dbbranch/daemon is down or the branch is
  unmanaged.
- Detached HEAD checkout is a no-op.
- A pre-existing foreign hook is preserved (refused without `--force`, appended
  with markers under `--force`).

## Validation
- Integration on macOS CI: install hook in a temp repo, perform `git switch`,
  assert the active branch changed; test detached HEAD, uninitialized project,
  and daemon-down cases (git must not hang or error).

## Dependencies
- 11 (`--if-managed`), 12 (auto-start).

## Non-goals
- Automatic HEAD-watching (v2).

## Design References
- [ADR-007](../decisions/ADR-007-cli-first-optin-hook.md), [DESIGN §11.2](../DESIGN.md#112-abuse-cases--mitigations)
