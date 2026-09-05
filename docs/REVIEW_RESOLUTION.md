# Review resolution record

- Repository: `Saber5656/dbbranch`
- Pull request: #1
- Parent head observed before this addendum: `72f669511ca58cfc29442e3755b905747c300490`
- Scope: existing review threads only; no new Bot review is requested.
- This document records design-level resolutions and focused verification gates. It does not claim implementation or test completion.

## Thread `PRRT_kwDOTNkEBc6OeUK3`

### Use a collision-free branch directory encoding

- Finding: The existing review thread `PRRT_kwDOTNkEBc6OeUK3` identifies this contract gap.
- Normative resolution: Encode the complete branch name with an unambiguous byte/length or base64url scheme without pre-replacing separators, persist the original name beside the directory, and reject a directory whose recorded name does not match.
- Focused verification before resolving this thread: Generate names such as `feature/x` and `feature__x` plus long/unicode names and assert distinct validated directories with reversible recorded identity.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkEBc6OeUK6`

### Require auth on the proxy-facing socket

- Finding: The existing review thread `PRRT_kwDOTNkEBc6OeUK6` identifies this contract gap.
- Normative resolution: Require a per-project connection credential on the loopback proxy protocol, store it with restrictive permissions, and reject missing/invalid credentials before forwarding to the trusted PostgreSQL socket.
- Focused verification before resolving this thread: Connect from a second loopback client with no, wrong, and correct credentials and assert only the authenticated connection is forwarded.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkEBc6OeUK7`

### Do not execute repo seed SQL with psql as trusted input

- Finding: The existing review thread `PRRT_kwDOTNkEBc6OeUK7` identifies this contract gap.
- Normative resolution: Treat repository seed SQL as untrusted: never feed it to a superuser `psql` session; use a dedicated non-superuser role with disabled shell-capable privileges, reject client meta-commands, and apply statement/time/resource limits.
- Focused verification before resolving this thread: Use seed SQL containing a psql shell escape and `COPY PROGRAM` against the fixture database and assert it is rejected or cannot execute outside the seed contract.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkEBc6OeUK8`

### Reject or clone external tablespaces

- Finding: The existing review thread `PRRT_kwDOTNkEBc6OeUK8` identifies this contract gap.
- Normative resolution: At branch-clone time detect `pg_tblspc` links outside the branch root and reject the operation with an actionable error in v1; cloning external targets is deferred until it can preserve isolation.
- Focused verification before resolving this thread: Create a cluster with an external tablespace and assert branch creation refuses it without sharing or mutating the target.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkEBc6OeUK9`

### Cap branch socket paths below macOS AF_UNIX limits

- Finding: The existing review thread `PRRT_kwDOTNkEBc6OeUK9` identifies this contract gap.
- Normative resolution: Validate the complete socket path, including project/run prefix and `.s.PGSQL.<port>`, against the macOS AF_UNIX byte limit; shorten/reject the encoded directory before `pg_ctl` starts.
- Focused verification before resolving this thread: Use a long valid branch name and assert the final path is within the platform limit and startup does not fail due to path length.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkEBc6OeUK-`

### Prevent max_instances=1 from blocking all branch switches

- Finding: The existing review thread `PRRT_kwDOTNkEBc6OeUK-` identifies this contract gap.
- Normative resolution: During a switch, treat the current active instance as an eviction candidate after the next branch clone is prepared; stop it before starting the new one, then commit active-branch state transactionally.
- Focused verification before resolving this thread: Switch A→B with `max_instances=1` and assert B starts, A is stopped, and no `too_many_instances` error is returned.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkEBc6OeUK_`

### Leave the base branch absent until its clone exists

- Finding: The existing review thread `PRRT_kwDOTNkEBc6OeUK_` identifies this contract gap.
- Normative resolution: Make init clone the base PGDATA before recording a stopped base branch; if cloning fails, leave the branch record absent so the normal create path remains valid.
- Focused verification before resolving this thread: Inject a clone failure during init and assert no phantom base record exists; on success assert the record points to an existing clone.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkEBc6OeULA`

### Avoid repo-controlled PostgreSQL binaries on PATH

- Finding: The existing review thread `PRRT_kwDOTNkEBc6OeULA` identifies this contract gap.
- Normative resolution: Resolve PostgreSQL binaries from a trusted configured directory or known system install locations, reject repo-controlled/writable PATH hits unless explicitly pinned by the user, and verify the resolved executable location.
- Focused verification before resolving this thread: Place fake `postgres`/`psql` binaries in the repository and PATH, then assert they are rejected and never executed.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkEBc6OeULC`

### Preserve last-checkout-wins semantics in the hook

- Finding: The existing review thread `PRRT_kwDOTNkEBc6OeULC` identifies this contract gap.
- Normative resolution: Attach an explicit target branch, checkout generation, and current-HEAD identity to asynchronous hook requests; the daemon must discard stale completions before changing active state.
- Focused verification before resolving this thread: Issue rapid A→B→C hook requests with delayed completions and assert only C can commit the active branch.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkEBc6OeULD`

### Fsync the state directory after rename

- Finding: The existing review thread `PRRT_kwDOTNkEBc6OeULD` identifies this contract gap.
- Normative resolution: After writing and fsyncing the temporary state file, rename it and fsync the containing state directory before reporting success; document platform error handling.
- Focused verification before resolving this thread: Run the persistence fault-injection/ordering test and assert the directory sync occurs after rename and failures are surfaced.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkEBc6OeULE`

### Do not repoint the proxy before state persists

- Finding: The existing review thread `PRRT_kwDOTNkEBc6OeULE` identifies this contract gap.
- Normative resolution: Persist the new active branch first, then repoint the proxy; if proxy update fails, restore the prior durable state or perform the defined rollback so state and routing cannot diverge.
- Focused verification before resolving this thread: Inject persistence and proxy failures separately and assert failed switches leave both the durable active branch and proxy target consistent.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Bot review policy

The existing Bot review is not re-triggered for this PR. Replies and thread resolution are performed only after the focused verification conditions above are recorded.