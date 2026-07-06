# dbbranch — Issue Plan (v1)

Canonical roadmap derived from [`DESIGN.md`](DESIGN.md). GitHub Issues are
generated from the per-issue drafts in [`issues/`](issues/); if they disagree,
this plan and the drafts win.

## v1 completion statement

> When every issue below (01–25) is implemented and validated, dbbranch v1 is
> complete: on macOS/APFS, a developer can `dbbranch init` a PostgreSQL-backed
> project, get a **stable connection string** that never changes, and create /
> switch / delete **instant, isolated per-branch databases** (manually or via an
> opt-in `post-checkout` hook), served through a loopback-only proxy by a
> self-managing daemon, with the security guarantees in
> [DESIGN §11](DESIGN.md#11-security-model), installable via Homebrew — except for
> newly discovered implementation unknowns (see "Known unknowns").

## Issue list (recommended execution order)

| # | Issue | Wave |
|---|---|---|
| 01 | [Repo scaffolding & CLI skeleton](issues/01-repo-scaffolding-cli-skeleton.md) | 0 |
| 02 | [Committed config parser & validation](issues/02-committed-config-parser.md) | 0 |
| 03 | [Local state store](issues/03-local-state-store.md) | 0 |
| 04 | [Project identity, home layout & branch encoding](issues/04-project-identity-home-layout.md) | 0 |
| 05 | [PostgreSQL detection & version compat](issues/05-postgres-detection-version-compat.md) | 0 |
| 07 | [PostgreSQL instance manager](issues/07-postgres-instance-manager.md) | 1 |
| 08 | [APFS CoW clone engine](issues/08-cow-clone-engine.md) | 1 |
| 20 | [Security: loopback, perms, pg_hba](issues/20-security-loopback-perms-hba.md) | 1 |
| 06 | [`init` & base cluster](issues/06-init-command-base-cluster.md) | 2 |
| 09 | [Branch CRUD](issues/09-branch-crud-commands.md) | 2 |
| 10 | [Loopback TCP proxy with draining](issues/10-tcp-proxy-with-draining.md) | 2 |
| 12 | [Daemon lifecycle & control socket](issues/12-daemon-lifecycle-control-socket.md) | 3 |
| 11 | [`switch` activation](issues/11-switch-command-activation.md) | 3 |
| 13 | [Daemon RPC handlers](issues/13-daemon-rpc-handlers.md) | 3 |
| 14 | [Daemon reconciliation & recovery](issues/14-daemon-reconciliation-recovery.md) | 4 |
| 15 | [Idle-stop, caps, `gc`](issues/15-idle-stop-gc-limits.md) | 4 |
| 16 | [`status`, `url`, `connection`](issues/16-status-url-connection-commands.md) | 4 |
| 17 | [Git post-checkout hook](issues/17-git-post-checkout-hook.md) | 4 |
| 18 | [`base import` & refresh](issues/18-base-import-refresh.md) | 4 |
| 19 | [Logging, errors & `doctor`](issues/19-logging-errors-doctor.md) | 4 |
| 21 | [Security: untrusted repo config & fuzzing](issues/21-security-untrusted-repo-config.md) | 5 |
| 22 | [Security: secret handling & redaction](issues/22-security-secret-handling.md) | 5 |
| 25 | [E2E integration harness](issues/25-e2e-integration-harness.md) | 5 |
| 23 | [Packaging: Homebrew, release](issues/23-packaging-homebrew-release.md) | 6 |
| 24 | [Docs: README, SECURITY, CONTRIBUTING](issues/24-docs-readme-security.md) | 6 |

## Dependency table

| Issue | Depends on |
|---|---|
| 01 | — |
| 02 | 01 |
| 03 | 01 |
| 04 | 01 |
| 05 | 01, 02 |
| 07 | 01, 04, 05, (20 for config gen) |
| 08 | 01, 04, 07 |
| 20 | 04, 06, 07, 10 |
| 06 | 02, 03, 04, 05, 07, 20, (17 optional) |
| 09 | 03, 04, 07, 08, (11 for reset-of-active) |
| 10 | 01, 07 |
| 12 | 01, 03, 04, 05, 10 |
| 11 | 03, 04, 07, 08, 09, 10 (invoked via 12/13) |
| 13 | 12, 06, 09, 11, 15, 18 |
| 14 | 03, 04, 07, 10, 12 |
| 15 | 03, 07, 10, 12, 14 |
| 16 | 12, 13 |
| 17 | 11, 12 |
| 18 | 03, 06, 07, 09 |
| 19 | 01, 04, 05, 08, 12 |
| 21 | 02, 04, 05, 20 |
| 22 | 19, 20 |
| 23 | 01 |
| 24 | 06, 09, 11, 16, 17, 20, 21, 22 |
| 25 | 06, 07, 08, 09, 10, 11, 12, 13, 14, 15, 16, 17, 20, 21 |

> Note: 20 and the instance/init issues are mutually referential (20 provides the
> `pg_hba`/`postgresql.auto.conf` generator that 06/07 consume, while 20's bind
> and perms self-tests exercise 07/10). Implement 20's **config-generation helper
> and perms helpers first** (needed by 06/07), then complete 20's proxy/instance
> self-tests after 10 lands. This is called out in issue 20's Scope.

## Dependency graph (waves)

```
Wave 0 (foundations, parallelizable after 01):
  01 ─┬─ 02 ─┐
      ├─ 03  │
      ├─ 04  │
      └─ 05 ─┘   (05 also needs 02)

Wave 1 (mechanisms):
  04,05 ─▶ 07 ─▶ 08
  (20: perms + pg config-gen helpers land here for 06/07)

Wave 2 (surfaces on mechanisms):
  06 (needs 02,03,04,05,07,20)
  09 (needs 03,04,07,08)
  10 (needs 07)

Wave 3 (daemon & activation):
  12 (needs 03,04,05,10) ─▶ 11 ─▶ 13

Wave 4 (operations & UX):
  14, 15 (need 12,14) , 16, 17, 18, 19

Wave 5 (security hardening & E2E):
  21, 22, 25

Wave 6 (release):
  23, 24
```

## Implementation waves

| Wave | Theme | Issues | Exit criterion |
|---|---|---|---|
| 0 | Foundations | 01, 02, 03, 04, 05 | `make ci` green; config/state/paths/detection unit-tested |
| 1 | Core mechanisms | 07, 08, (20 helpers) | Boot a real instance; clone a stopped base and boot the clone |
| 2 | Command surfaces | 06, 09, 10 | `init` a project; create/list/delete branches; proxy reaches an instance |
| 3 | Daemon & switch | 12, 11, 13 | End-to-end `switch` through the daemon changes the active DB |
| 4 | Operations & UX | 14, 15, 16, 17, 18, 19 | Recovery, gc/idle, status/url, hook, base import, doctor all work |
| 5 | Security & E2E | 20 (complete), 21, 22, 25 | Threat-model tests + full E2E suite green |
| 6 | Release | 23, 24 | Homebrew install works; docs complete; ready to publish (not merged/released here) |

## Coverage: DESIGN.md section → issue(s)

| DESIGN section | Issue(s) |
|---|---|
| §1 Product summary / §1.1 user story | 06, 11, 16, 24 |
| §4 Architecture / components | 01, 07, 08, 10, 12 |
| §5.1 Directory layout | 04, 06 |
| §5.2 Branch↔dir encoding | 04 |
| §5.3 Project identity | 04 |
| §5.4 Committed config schema | 02, 21 |
| §5.5 Trusted local config | 05 |
| §5.6 state.json | 03 |
| §6.1 Per-branch state machine | 03, 09, 11 |
| §6.2 Switch sequence | 11 |
| §7 Daemon | 12, 13, 14, 15 |
| §7.2 Control protocol | 12, 13 |
| §7.5 Reconciliation | 14 |
| §8 PostgreSQL detection/instances | 05, 07 |
| §8.3 Version invariant | 05, 19 |
| §9 Clone engine | 08 |
| §9.2 Consistency (ADR-005) | 08, 06, 18 |
| §10 Proxy | 10 |
| §11 Security model | 20, 21, 22, 24 |
| §11.1 Trust boundaries | 20, 21, 22 |
| §11.2 Abuse cases | 15, 17, 20, 21, 22 |
| §11.3 Secure defaults | 20, 22 |
| §12 CLI surface | 06, 09, 11, 15, 16, 17, 18, 19 |
| §13 Failure modes | 14, 19, 25 |
| §14 v2 deferred | (none — out of v1 scope) |
| §15 Known unknowns | 08, 25 (+ below) |

Every v1 DESIGN section maps to at least one issue. §14 is intentionally
unmapped (deferred). This satisfies the completion statement.

## Validation strategy (whole product)

1. **Unit tests** per package (issues 02–05, 19, 22): config validation, state
   round-trips, path/encoding, detection, redaction — table-driven + property
   tests.
2. **Fuzzing** (issue 21): TOML loader and `sql_path` resolver against adversarial
   corpora; short fuzz in CI, longer documented.
3. **Integration tests** with real PostgreSQL on macOS CI (issues 07–18, 20):
   instance lifecycle, cloning, proxy draining, daemon RPC, reconciliation,
   idle/gc, base import.
4. **Security tests** (issues 20–22): loopback-only + no-instance-TCP assertions,
   perms assertions, malicious-repo inertness, secret-leak scans.
5. **End-to-end** (issue 25): the full user workflow incl. hook, crash recovery,
   isolation, instant-clone timing, and security assertions — gating v1 release.
6. **Docs/link/markdown lint** (issue 24) in CI.
7. **CI matrix**: `macos-latest`, `make ci` + `make e2e` (e2e installs PostgreSQL
   via Homebrew). Release workflow on `v*` tags (issue 23).

## Deferred v2 items

- Zero-downtime clone-from-running via `pg_backup_start`/`pg_backup_stop`
  ([ADR-005](decisions/ADR-005-clone-consistency-strategy.md)).
- Linux (ZFS/Btrfs/overlayfs) + a storage-engine abstraction
  ([ADR-004](decisions/ADR-004-macos-apfs-only-v1.md)).
- MySQL / SQLite engines behind that abstraction.
- HEAD-watching daemon mode (fully automatic, no hook)
  ([ADR-007](decisions/ADR-007-cli-first-optin-hook.md)).
- Snapshots / time-travel / named per-branch checkpoints.
- Automated `pg_upgrade` of base across PostgreSQL majors.
- Optional password/TLS auth (issue 22 leaves a guarded seam).
- `CREATE DATABASE ... TEMPLATE` fast-path for tiny databases.
- Notarization/signing automation; non-macOS packaging.

## Known unknowns (may create additional issues during implementation)

1. Exact Go binding for `copyfile(3)` `COPYFILE_CLONE_FORCE` and its recursive
   behavior over `pg_wal`/tablespace symlinks (issue 08 spike may split out a
   dedicated binding issue).
2. `COPYFILE_CLONE_FORCE` handling of non-regular files if any survive in a
   stopped `PGDATA` (sockets/FIFOs) — verify; may need a pre-clone sanitizer step.
3. macOS per-process fd/semaphore limits with many concurrent instances — may
   force a lower default `max_instances` or a limit-raising step (issue 15).
4. Whether L4-only proxying is fully transparent to all client drivers'
   startup/SSL/reconnection behavior — validate in issue 25; a protocol-aware
   proxy would become a new issue if not.
5. APFS local-snapshot / Time Machine interaction with heavy branch churn and
   space accounting for CoW-shared blocks (affects `status` disk reporting).
6. `pg_ctl` fast-shutdown latency under load impacting switch/idle timing budgets.
7. Behavior when the user upgrades PostgreSQL major mid-project (base becomes
   unbootable) — v1 detects and explains; a guided `pg_upgrade`/recreate flow may
   be pulled forward from v2 if it proves common.

New issues discovered here must be added to `issues/` and this plan **before**
their GitHub Issues are created.
