# dbbranch — Design (v1)

> Give every Git branch its own instant, isolated local PostgreSQL database.
> A "local Neon" for developers, built on macOS APFS copy-on-write clones.

This document is the **canonical low-level design** for dbbranch v1. It is the
source of truth. GitHub Issues and Pull Requests are derived artifacts; if they
disagree with this file, this file wins and they are stale.

Audience: implementation agents of modest capability. The design is written so
that the issues in [`ISSUE_PLAN.md`](ISSUE_PLAN.md) can be executed mechanically,
with explicit schemas, state machines, commands, file layouts, and failure modes.

---

## 1. Product summary

`dbbranch` is a macOS command-line tool that maps **Git branches to isolated
PostgreSQL databases**. Switching a database branch is near-instant regardless of
data size because each branch's PostgreSQL data directory (`PGDATA`) is an APFS
copy-on-write clone of a base snapshot, and clones diverge lazily.

The developer's application connects to a **single, stable connection string**
(a fixed loopback TCP port). dbbranch runs a lightweight TCP proxy on that port
and routes new connections to whichever branch is currently active. The
application's configuration never changes when the branch changes.

### 1.1 Primary user story

```
$ dbbranch init                       # once per project
$ export DATABASE_URL=$(dbbranch url) # stable; never changes again
$ git switch -c feature/checkout
$ dbbranch switch feature/checkout    # instant isolated DB, cloned from base
#   ... run migrations, mangle data, break things freely ...
$ git switch main
$ dbbranch switch main                # back to a clean, untouched main DB
```

With the opt-in Git hook installed, the two `dbbranch switch` calls happen
automatically on `git switch`/`git checkout`.

### 1.2 Value proposition

- **Instant**: branch creation is O(1) in data size (APFS clone), like Neon.
- **Isolated**: destructive migrations and seed data never leak across branches.
- **Zero app reconfiguration**: one stable connection string via the proxy.
- **Local & private**: no cloud, no account, data never leaves the machine.

---

## 2. Scope

### 2.1 v1 scope (in)

- macOS only (Apple Silicon and Intel), APFS-backed volume required.
- PostgreSQL only. Uses the user's **already-installed** PostgreSQL binaries.
- APFS copy-on-write clone of `PGDATA` + one PostgreSQL instance per branch.
- Fixed loopback-only TCP proxy for a stable connection string.
- CLI-first UX; opt-in `post-checkout` Git hook.
- Background daemon (`dbbranchd`) that owns instances, the proxy, and state.
- Base snapshot initialized empty or seeded from a SQL dump.
- Branch lifecycle: create, switch, list, delete, reset, gc.
- Security posture for a local developer tool, treated as a first-class concern.
- Homebrew-tap distribution.

### 2.2 v1 non-goals (out)

- Linux and Windows support (see [ADR-004](decisions/ADR-004-macos-apfs-only-v1.md)).
- MySQL, SQLite, or any non-PostgreSQL engine.
- Non-APFS filesystems (ZFS/Btrfs/overlayfs) and dump/restore fallbacks.
- Bundling or downloading PostgreSQL binaries (see [ADR-003](decisions/ADR-003-user-provided-postgres.md)).
- Automatic HEAD-watching daemon that switches without a CLI/hook trigger.
- Page-level / WAL-pointer branching à la Neon (we clone whole clusters).
- Multi-user, networked, or shared-server operation. Loopback only.
- Production database management, HA, replication, or backups-as-a-product.
- A GUI.

### 2.3 v2 / deferred (see §14)

Zero-downtime clone via the low-level backup API, additional filesystems, other
engines behind a storage abstraction, HEAD-watching mode, snapshot/time-travel.

---

## 3. Key decisions (ADR index)

| ADR | Decision |
|---|---|
| [ADR-001](decisions/ADR-001-cow-clone-per-branch-instance.md) | APFS copy-on-write clone of `PGDATA` + one PostgreSQL instance per branch |
| [ADR-002](decisions/ADR-002-fixed-port-l4-proxy.md) | Fixed loopback port + L4 (byte-stream) proxy for a stable connection string |
| [ADR-003](decisions/ADR-003-user-provided-postgres.md) | Use the user's installed PostgreSQL; never bundle/download server binaries |
| [ADR-004](decisions/ADR-004-macos-apfs-only-v1.md) | macOS + APFS only for v1 |
| [ADR-005](decisions/ADR-005-clone-consistency-strategy.md) | v1 clone consistency via source quiesce; low-level backup API deferred |
| [ADR-006](decisions/ADR-006-unix-socket-instances-loopback-proxy.md) | Instances on Unix sockets; only the proxy binds loopback TCP |
| [ADR-007](decisions/ADR-007-cli-first-optin-hook.md) | CLI-first activation; Git hook is opt-in and non-blocking |

Comparative and background research: [`research/`](research/).

---

## 4. Architecture overview

```
                    ┌──────────────────────────────────────────────────────┐
   git / user       │                    dbbranch home                     │
      │             │        ~/.dbbranch/projects/<project-id>/            │
      ▼             │                                                      │
 ┌─────────┐  ctrl  │  ┌──────────────┐   manages   ┌───────────────────┐  │
 │  CLI    │──sock──┼─▶│  dbbranchd   │────────────▶│ postgres instance │  │
 │dbbranch │        │  │  (daemon)    │             │  branch: main     │  │
 └─────────┘        │  │              │             │  unix socket      │  │
                    │  │  - state     │             └───────────────────┘  │
 ┌─────────┐        │  │  - proxy     │             ┌───────────────────┐  │
 │post-    │──ctrl──┼─▶│  - lifecycle │────────────▶│ postgres instance │  │
 │checkout │  sock  │  │  - reconcile │             │  branch: feature  │  │
 │  hook   │        │  └──────┬───────┘             │  unix socket      │  │
 └─────────┘        │         │ routes new conns    └───────────────────┘  │
                    │         ▼                                            │
 ┌─────────┐  TCP   │  ┌──────────────┐                                    │
 │  app    │───────▶│  │ proxy        │  fixed loopback port (127.0.0.1)   │
 │DATABASE_│  5432- │  │ 127.0.0.1:N  │  → active branch's unix socket     │
 │  URL    │  like  │  └──────────────┘                                    │
 └─────────┘        └──────────────────────────────────────────────────────┘
                     APFS volume: branch PGDATAs are CoW clones of base
```

### 4.1 Components

| Component | Binary / module | Responsibility |
|---|---|---|
| CLI | `dbbranch` | Parse commands, talk to the daemon over its control socket, render output. Owns almost no state. |
| Daemon | `dbbranchd` | Long-running per-user process. Owns: state store, PostgreSQL instance lifecycle, the TCP proxy, reconciliation, idle-stop, gc. Single writer of state. |
| Control plane | Unix domain socket + line-delimited JSON RPC | CLI↔daemon and hook↔daemon transport. Same-user only, filesystem-permission authenticated. |
| Clone engine | `internal/clone` | APFS CoW clone of a `PGDATA` directory via `copyfile(3)` `COPYFILE_CLONE_FORCE`. |
| Instance manager | `internal/pg` | Start/stop/health a single PostgreSQL instance on a private Unix socket via `pg_ctl`/`postgres`. |
| Proxy | `internal/proxy` | L4 byte-stream proxy on the fixed loopback port; routes new connections to the active instance; drains old connections. |
| Config | `internal/config` | Parse & validate committed `.dbbranch/config.toml` (untrusted) and local user config (trusted). |
| State store | `internal/state` | Atomic, locked JSON state per project. |
| Postgres detect | `internal/pgfind` | Locate & validate `initdb`/`postgres`/`pg_ctl`, parse version. |

The daemon is a **single per-user process** that can serve multiple projects.
Each project gets its own fixed proxy port, its own set of instances, and its own
state file. Projects are isolated from each other.

---

## 5. On-disk layout & data model

### 5.1 Directory layout

Machine-local state (never committed, outside the repo):

```
~/.dbbranch/                              (0700)
├── config.toml                           (0600)  user/global trusted config
├── daemon.sock                           (0600)  daemon control socket
├── daemon.pid                            (0600)
├── daemon.log
└── projects/
    └── <project-id>/                     (0700)  project-id = see §5.3
        ├── project.json                  (0600)  resolved project record
        ├── state.json                    (0600)  authoritative branch state
        ├── state.lock                            advisory lock file
        ├── base/
        │   └── pgdata/                    (0700)  canonical base cluster (quiesced)
        ├── branches/
        │   ├── main/
        │   │   ├── pgdata/                (0700)  CoW clone of base/pgdata
        │   │   └── instance.pid
        │   └── feature__checkout/
        │       └── pgdata/                (0700)
        ├── run/
        │   ├── main/.s.PGSQL.5432         (0700 dir) per-branch unix socket
        │   └── feature__checkout/...
        └── logs/
            ├── main.log
            └── feature__checkout.log
```

Repo-committed, shareable config (in the project working tree):

```
<repo-root>/.dbbranch/config.toml         committed; UNTRUSTED at parse time
<repo-root>/.dbbranch/seed.sql            optional committed seed (UNTRUSTED)
```

> `.dbbranch/` in the repo holds only **shareable, declarative** settings. It
> must never be able to make dbbranch run an arbitrary binary or write outside
> the project's dbbranch home. See §11 and [issue 21](issues/21-security-untrusted-repo-config.md).

### 5.2 Branch name → directory name mapping

Git branch names may contain `/`, which cannot be a directory component here.

- Encoding: replace every `/` with `__` (double underscore), then percent-encode
  any character outside `[A-Za-z0-9._-]` as `%XX` (uppercase hex).
- The mapping is bijective and stored explicitly in `state.json` (`dirName`
  field) so decoding never relies on reversing the transform.
- Reserved: a branch whose encoded name is `base`, `run`, `logs`, `.`, or `..`
  is rejected with a clear error.
- Max encoded length 200 bytes; longer names are rejected.

### 5.3 Project identity

- `project-id = hex(sha256(canonical-absolute-repo-root))[:16]`.
- `canonical-absolute-repo-root` = `realpath` of the output of
  `git rev-parse --show-toplevel`, run from the invocation directory.
- The full canonical path is stored in `project.json` for display and collision
  detection. If a stored path differs from the recomputed one for the same id
  (hash collision, astronomically unlikely), the daemon refuses and logs.

### 5.4 `config.toml` (committed, UNTRUSTED) schema

```toml
version = 1                     # required int; must equal 1 in v1
postgres_version = "16"         # required string; major version constraint, "^[0-9]{1,2}$"
database = "app"                # required; app database name created in base, "^[A-Za-z_][A-Za-z0-9_]{0,62}$"
base_branch = "main"           # optional; default clone source, valid git branch name
proxy_port = 54320             # optional; preferred loopback port, 1024..65535
idle_timeout = "10m"           # optional; Go duration; "0" disables idle-stop
max_instances = 5              # optional; 1..64; concurrent running instances cap

[seed]
strategy = "empty"             # "empty" | "sql"
sql_path = ".dbbranch/seed.sql" # required iff strategy="sql"; must be repo-relative, no "..", no leading "/"
```

**Untrusted-field rules** (enforced by the parser, issue 02 + 21):

- Unknown keys → hard error (no silent ignore).
- `proxy_port` must be in `1024..65535`. Ports `<1024` rejected.
- `postgres_version` is a **constraint only**, matched against the locally
  detected binary's major version. It is never a path and never selects a binary.
- `sql_path` must be a normalized repo-relative path with no `..` segment, no
  absolute prefix, and must resolve inside the repo working tree after
  `filepath.Clean`. Symlinks in the resolved path that escape the repo → reject.
- There is **no field that names a binary, a shell command, or an absolute
  path**. The PostgreSQL binary location comes only from detection (§8) or from
  trusted local config (`~/.dbbranch/config.toml`), never from repo config.

### 5.5 Local/global trusted config `~/.dbbranch/config.toml` schema

```toml
version = 1
postgres_bin_dir = "/opt/homebrew/opt/postgresql@16/bin"  # optional TRUSTED override
daemon_idle_shutdown = "30m"     # daemon self-exit when no projects active; "0" disables
log_level = "info"               # "error"|"warn"|"info"|"debug"
```

Only this trusted file may specify `postgres_bin_dir` (an absolute path to a
binary directory). Repo config cannot.

### 5.6 `state.json` schema (authoritative, daemon-owned)

```jsonc
{
  "schemaVersion": 1,
  "projectId": "a1b2c3d4e5f60718",
  "repoRoot": "/Users/dev/myapp",
  "createdAt": "2026-07-06T09:00:00Z",
  "postgres": {
    "binDir": "/opt/homebrew/opt/postgresql@16/bin",
    "majorVersion": 16,
    "fullVersion": "16.3"
  },
  "proxy": {
    "port": 54320,
    "listenAddr": "127.0.0.1:54320",
    "activeBranch": "main"          // null if none active
  },
  "baseBranch": "main",
  "database": "app",
  "branches": {
    "main": {
      "name": "main",               // decoded git branch name
      "dirName": "main",            // §5.2 encoding
      "state": "running",           // §6.1 states
      "createdFrom": "base",        // "base" | "<branch>"
      "pgdataPath": ".../branches/main/pgdata",
      "socketDir": ".../run/main",
      "pid": 41234,                 // null when not running
      "lastActiveAt": "2026-07-06T09:05:00Z",
      "clonedAt": "2026-07-06T09:00:10Z"
    }
  }
}
```

- Writes are atomic: write to `state.json.tmp`, `fsync`, `rename`. The daemon
  holds `state.lock` (advisory `flock`) for the duration of a mutation.
- `schemaVersion` gates forward migrations. Unknown newer versions → daemon
  refuses to run and asks the user to upgrade.

---

## 6. Branch lifecycle & state machines

### 6.1 Per-branch state machine

States: `absent` → `cloning` → `stopped` → `starting` → `running` →
`(active)` → `stopping` → `stopped`; plus `error`.

`active` is not a branch state but a **project-level pointer** (`proxy.activeBranch`);
at most one branch is active per project. A branch must be `running` to become
active.

```
absent ──create──▶ cloning ──ok──▶ stopped ──start──▶ starting ──ready──▶ running
   ▲                  │ fail          │                   │ fail            │
   │                  ▼               │                   ▼                 │
   └──────────────── error ◀──────────┴───────────────── error             │
                                                                           │
running ──idle-timeout / stop──▶ stopping ──▶ stopped                       │
running ──switch(this)──▶ running + activeBranch=this  ◀────────────────────┘
delete: any non-active state ──▶ stopping? ──▶ removing clone ──▶ absent
```

Transition table:

| From | Event | Action | To |
|---|---|---|---|
| absent | `create`/`switch` | quiesce source, CoW clone base/parent → pgdata | cloning |
| cloning | clone ok | write branch record | stopped |
| cloning | clone fail | remove partial clone | error/absent |
| stopped | `start`/`switch` | `pg_ctl start` on private socket | starting |
| starting | readiness ok (`pg_isready`) | record pid | running |
| starting | timeout/exit | capture logs | error |
| running | `switch <this>` | set `activeBranch=this`, repoint proxy target | running (active) |
| running | idle ≥ timeout & not active & 0 conns | `pg_ctl stop -m fast` | stopping |
| running | `stop` | `pg_ctl stop -m fast` | stopping |
| stopping | stopped | clear pid | stopped |
| any | `delete` (not active) | stop if running, then `rm -rf` clone dir | absent |
| running | postgres crash detected | mark, capture logs | error |
| error | `reset`/`recreate` | remove clone, re-clone | cloning |

### 6.2 Project-level switch sequence

`dbbranch switch <branch>`:

1. CLI resolves target branch. If `<branch>` omitted, uses current Git branch
   (`git symbolic-ref --short HEAD`). Detached HEAD → error (or no-op in hook).
2. CLI sends `switch{branch}` RPC to daemon (auto-starts daemon if down, §7.4).
3. Daemon acquires project lock.
4. If branch `absent`: choose clone source (`--from`, else `base_branch`);
   ensure source is quiesced (§9); CoW clone → `cloning`→`stopped`.
5. If branch `stopped`: start instance → `starting`→`running`.
6. Set `proxy.activeBranch = branch`; the proxy begins routing **new**
   connections to this branch's socket. Existing connections to the previously
   active instance keep their target and drain naturally (§10).
7. Persist state; release lock; return status to CLI.

Idempotent: switching to the already-active running branch is a no-op that still
returns success.

---

## 7. Daemon (`dbbranchd`)

### 7.1 Responsibilities

Single per-user process. Sole writer of `state.json`. Owns the proxy listeners,
the child PostgreSQL processes, idle-stop timers, gc, and crash reconciliation.

### 7.2 Control protocol

- Transport: Unix domain socket at `~/.dbbranch/daemon.sock`, mode `0600`,
  parent dir `0700`. No TCP control plane ([ADR-006](decisions/ADR-006-unix-socket-instances-loopback-proxy.md)).
- Framing: newline-delimited JSON. One request object per line, one response
  object per line.
- Request: `{"id": "<uuid>", "method": "<name>", "params": { ... }}`.
- Response: `{"id": "<uuid>", "ok": true, "result": {...}}` or
  `{"id": "<uuid>", "ok": false, "error": {"code":"...","message":"..."}}`.
- Every request carries `projectRoot` (absolute) so the daemon selects the
  project; the daemon re-canonicalizes and re-derives the project-id itself
  (never trusts a client-supplied project-id).

Methods (v1): `ping`, `init`, `status`, `list`, `switch`, `create`, `delete`,
`reset`, `stop`, `gc`, `baseImport`, `shutdown`.

### 7.3 Concurrency & locking

- One in-process mutex per project serializes mutations.
- `state.lock` (flock) guards against a second daemon (should not happen; see
  single-instance lock) and documents intent.
- Single-instance daemon: `daemon.pid` + flock on it; a second `dbbranchd`
  start detects the live daemon and exits 0.

### 7.4 Auto-start & shutdown

- CLI, on connection refused to `daemon.sock`, spawns `dbbranchd` (double-fork,
  detached), waits for `ping` up to 5s, then proceeds.
- Daemon self-exits after `daemon_idle_shutdown` with no running instances
  across all projects (default 30m; `0` disables). `dbbranch daemon stop` forces
  it (stops all instances first).

### 7.5 Reconciliation (crash recovery)

On startup the daemon, for each known project:

1. Load `state.json`.
2. For each branch with a recorded `pid`: check the pid is alive **and** is a
   PostgreSQL process for this branch's `pgdata` (match `postmaster.pid` in the
   data dir). If not, mark `stopped`, clear pid.
3. Remove stale sockets under `run/` for stopped branches.
4. Detect orphan `pgdata` dirs not in state → offer to `gc`.
5. `activeBranch` whose instance is not running → clear (proxy serves errors
   until next switch).

---

## 8. PostgreSQL detection & instance management

### 8.1 Detection (`internal/pgfind`, issue 05)

Search order for `initdb`, `postgres`, `pg_ctl`, `pg_isready`, `psql`:

1. `postgres_bin_dir` from **trusted** `~/.dbbranch/config.toml` (if set).
2. `PATH`.
3. Known Homebrew locations: `/opt/homebrew/opt/postgresql@NN/bin`,
   `/usr/local/opt/postgresql@NN/bin` (NN from committed `postgres_version`).

- Resolve to absolute paths; `realpath` them; store in state.
- Run `postgres --version`; parse `postgres (PostgreSQL) 16.3` → major 16.
- Compatibility: the detected **major** version must equal
  `config.postgres_version`. A cluster initialized by major X cannot be started
  by major Y; mismatch → hard error with remediation text.
- All four/five binaries must come from the **same** bin dir (same install).

### 8.2 Instance manager (`internal/pg`, issue 07)

- Init base: `initdb -D <base/pgdata> --encoding=UTF8 --locale=C -A trust`
  (auth rationale §11). Generate a minimal `postgresql.auto.conf`:
  ```
  listen_addresses = ''                 # no TCP at all; unix socket only
  unix_socket_directories = '<run/<branch>>'
  unix_socket_permissions = 0700
  port = 5432                            # nominal; socket path is what matters
  fsync = on
  ```
- Start: `pg_ctl -D <pgdata> -l <logs/<branch>.log> -o "-k <socketDir>" start`.
- Readiness: poll `pg_isready -h <socketDir> -p 5432` until ready or 30s timeout.
- Stop: `pg_ctl -D <pgdata> -m fast stop` (checkpoints, then exits).
- Never expose TCP from an instance; only the proxy listens on TCP (§10, §11).
- Capture stderr/stdout to `logs/<branch>.log`; surface tail on error.

### 8.3 Version-consistency invariant

Base and all clones share one PostgreSQL major version (they are byte clones).
If the user upgrades PostgreSQL to a new major, the existing base is unusable by
the new binary; `dbbranch doctor`/`status` detects and explains
(`pg_upgrade`/recreate-base is a documented manual path; automation is v2).

---

## 9. Clone engine (`internal/clone`, issue 08)

### 9.1 Primitive

- Use `copyfile(3)` with `COPYFILE_CLONE_FORCE | COPYFILE_RECURSIVE` (Go: call
  via `golang.org/x/sys/unix`-style syscall wrapper or cgo `copyfile`), **not**
  raw `clonefile(2)` on directories — Apple explicitly discourages directory
  cloning through `clonefile(2)` and recommends `copyfile(3)`
  ([research](research/apfs-clonefile-and-consistency.md)).
- `COPYFILE_CLONE_FORCE` fails (rather than silently deep-copying) when a true
  clone is impossible; we treat that as a hard, explained error (e.g. source and
  dest on different volumes, or non-APFS).
- Preflight: verify source and destination resolve to the **same APFS volume**
  (same `st_dev`; `statfs` type `apfs`). Cross-device or non-APFS → actionable
  error ([ADR-004](decisions/ADR-004-macos-apfs-only-v1.md)).

### 9.2 Consistency ([ADR-005](decisions/ADR-005-clone-consistency-strategy.md))

- **Clone-from-base**: base is always **quiesced** (its instance is stopped when
  not the active/serving one, and is stopped before it is used as a clone
  source). A clone of a cleanly-stopped cluster is trivially consistent.
- **Clone-from-running-branch** (`--from <live-branch>`): v1 strategy is
  *quiesce-then-clone*: `pg_ctl stop -m fast` the source, clone, restart the
  source. Brief interruption on that source only. Zero-downtime via
  `pg_backup_start`/`pg_backup_stop` (PostgreSQL ≥15 names) is a documented v2
  optimization.
- After cloning a running/crashed cluster, PostgreSQL performs crash recovery on
  first start; we allow and wait for it. Because v1 always clones a quiesced
  source, recovery is a clean no-op in the normal path.
- Post-clone hygiene: delete `postmaster.pid` from the clone before first start
  (stale pid from a copied running dir would block startup).

### 9.3 Failure modes

| Failure | Detection | Handling |
|---|---|---|
| Non-APFS / cross-volume | `statfs`/`COPYFILE_CLONE_FORCE` fails | Refuse with remediation |
| `ENOSPC` mid-clone (metadata) | copyfile error | Remove partial clone, error |
| Source not quiesced | instance still running when required | Quiesce per §9.2 or refuse |
| Partial clone on crash | dir exists, no branch record | `gc` removes orphans |

---

## 10. Proxy (`internal/proxy`, issue 10)

- Listener: `127.0.0.1:<proxy_port>` only. Never `0.0.0.0`, never a non-loopback
  interface ([ADR-006](decisions/ADR-006-unix-socket-instances-loopback-proxy.md), issue 20).
- L4 byte-stream passthrough. On `accept`, read the **current**
  `activeBranch` target and dial that branch's Unix socket
  (`run/<branch>/.s.PGSQL.5432`). Splice bytes both directions until either side
  closes. The proxy does **not** parse the PostgreSQL wire protocol in v1 (lower
  attack surface, [ADR-002](decisions/ADR-002-fixed-port-l4-proxy.md)).
- Switch semantics (**drain**): changing `activeBranch` only affects
  connections accepted afterward. In-flight connections keep their original
  target and finish naturally; the previously active instance stays alive until
  it has zero proxied connections, after which it is eligible for idle-stop.
- No active branch / target socket absent: the proxy accepts then immediately
  closes, or refuses connect, surfacing a clear "no active branch — run
  `dbbranch switch`" condition (also reported by `status`).
- Per-project one listener. Bind failure (port in use) → actionable error at
  `init`/daemon start; suggest a different `proxy_port`.

---

## 11. Security model

Security is a v1 requirement, not a later hardening pass. Full threat model:
[issue 21](issues/21-security-untrusted-repo-config.md), [SECURITY.md](issues/24-docs-readme-security.md).

### 11.1 Trust boundaries

| Boundary | Trusted? | Control |
|---|---|---|
| Local user invoking CLI | Trusted | Runs as the user; owns all dirs |
| `~/.dbbranch/**` state & sockets | Trusted, integrity-protected | `0700`/`0600`; same-user only |
| Committed `.dbbranch/config.toml`, `seed.sql` | **UNTRUSTED** | Strict parse; no binary/command/path-escape (§5.4, issue 21) |
| Detected PostgreSQL binaries | Trusted toolchain | Absolute-resolved; version-checked; from PATH/known dirs/trusted config only |
| Proxy TCP port | Loopback only | `127.0.0.1`; never network-exposed |
| Instance sockets | Trusted | Unix socket in `0700` dir; `-A trust` limited to same-user local socket |
| Daemon control socket | Trusted | Unix socket `0600`; no remote control |
| Git hook | User-installed | Opt-in; minimal; calls resolved binary; non-blocking |

### 11.2 Abuse cases & mitigations

1. **Malicious repo config → RCE / path traversal**: a cloned repo ships
   `.dbbranch/config.toml`/`seed.sql`. Mitigation: config cannot name binaries,
   commands, or absolute paths; `sql_path` is confined to the repo tree with
   `..`/symlink-escape rejection; unknown keys rejected; numeric ranges enforced;
   seed SQL is executed **only** on explicit `init`/`base import`, never
   implicitly on `switch`, and only after the user has run dbbranch in that repo
   (informed action). Negative/fuzz tests in issue 21.
2. **Network exposure of dev DB**: instances never bind TCP
   (`listen_addresses=''`); the proxy binds `127.0.0.1` only; a startup assertion
   rejects any non-loopback bind (issue 20).
3. **PATH hijacking of `postgres`**: binaries resolved to absolute paths,
   `realpath`-ed, version-verified, and pinned in state; a changed resolved path
   is surfaced by `status`/`doctor`.
4. **Symlink attacks in state dir**: dbbranch creates its own dirs with `0700`,
   refuses to operate on state paths that are symlinks it did not create, and
   uses `O_NOFOLLOW` on sensitive opens where applicable.
5. **Secret leakage**: local `trust` auth means no password by default; if a
   password is ever generated it is stored `0600`, never logged, and redacted in
   all output (issue 22). State/logs live outside the repo and are never
   committed; `dbbranch init` writes `.dbbranch/` gitignore guidance for local
   artifacts if any were to land in-tree.
6. **Git hook hangs checkout**: hook runs with a short timeout, backgrounds the
   switch to the daemon, and always exits 0 so `git` is never blocked (issue 17).
7. **Resource exhaustion (many branches)**: `max_instances` cap, idle-stop,
   `dbbranch gc`, and disk-usage reporting in `status` (issues 15, 16).

### 11.3 Secure defaults

- Loopback-only everywhere; `0700`/`0600` perms on all created paths.
- `trust` auth restricted to the same-user local Unix socket; no TCP auth surface.
- No telemetry, no network calls in v1 (Homebrew handles updates out of band).
- Fail closed: ambiguous or invalid untrusted input is rejected, not guessed.

---

## 12. CLI surface (issue map)

| Command | Summary | Issue |
|---|---|---|
| `dbbranch init [--from-empty\|--from-dump <f>] [--install-hook]` | Initialize project: base cluster, config, state | 06 |
| `dbbranch status` | Current branch, active DB, connection string, per-branch states, disk | 16 |
| `dbbranch url` / `dbbranch connection` | Print the stable connection string | 16 |
| `dbbranch switch [<branch>] [--from <src>]` | Make branch active (clone+start as needed) | 11 |
| `dbbranch create <branch> [--from <src>]` | Create branch DB without switching | 09 |
| `dbbranch list` | List branch DBs and states | 09 |
| `dbbranch delete <branch>` | Stop + remove a branch DB | 09 |
| `dbbranch reset <branch>` | Re-clone a branch from base | 09 |
| `dbbranch gc` | Remove orphan clones/instances; enforce caps | 15 |
| `dbbranch base import <dump.sql>` | (Re)load base from a SQL dump | 18 |
| `dbbranch hook install\|uninstall` | Manage the `post-checkout` hook | 17 |
| `dbbranch daemon start\|stop\|status` | Manage the daemon | 12 |
| `dbbranch doctor` | Environment diagnostics (APFS, pg version, perms) | 19 |
| `dbbranch version` / `--version` | Version & build provenance | 01, 23 |

Global flags: `--json` (machine-readable output), `--quiet`, `--verbose`,
`--project <path>` (override repo root detection).

---

## 13. Failure modes & edge cases (cross-cutting)

| Situation | Behavior |
|---|---|
| Daemon down on CLI call | Auto-start; if start fails, actionable error |
| Not a git repo / no toplevel | `init` errors; other commands error clearly |
| Detached HEAD on `switch`/hook | `switch` errors; hook no-ops with a note |
| Branch name with `/`, unicode | Encoded per §5.2; mapping stored |
| PostgreSQL major mismatch vs base | Hard error; `doctor` explains |
| Non-APFS volume for dbbranch home | `init`/`doctor` refuse with remediation |
| Disk full during clone/start | Remove partial artifacts; error with cause |
| Port already in use | Actionable error; suggest changing `proxy_port` |
| Concurrent CLI invocations | Serialized by daemon per-project lock |
| Orphan instance after crash | Reconciled on daemon start (§7.5) |
| Deleting the active branch | Refused; must switch away first |
| Switching to branch whose parent clone was deleted | Falls back to `base_branch`; warns |
| Seed SQL fails during `init` | `init` aborts, removes half-built base, non-zero exit |

---

## 14. v2 / deferred ideas (not in v1)

- Zero-downtime clone-from-running via `pg_backup_start`/`pg_backup_stop`.
- Additional filesystems (ZFS/Btrfs) and a storage-engine abstraction.
- Additional DB engines (MySQL, SQLite) behind that abstraction.
- HEAD-watching daemon mode (fully automatic, no hook).
- Snapshots / time-travel / named checkpoints per branch.
- Automated `pg_upgrade` of base across PostgreSQL majors.
- Linux/Windows support.
- Multi-root / monorepo multiple-database projects.

---

## 15. Known unknowns (may spawn issues during implementation)

1. Exact Go binding for `copyfile(3)` `COPYFILE_CLONE_FORCE` (cgo vs syscall) and
   its recursive-clone behavior on nested `pg_wal`/tablespace links — validate in
   issue 08 spike; may split into a binding issue.
2. Behavior of `COPYFILE_CLONE_FORCE` when `PGDATA` contains sockets or FIFOs
   (should not, since we clone a stopped cluster, but verify).
3. macOS per-process open-file / semaphore limits with many concurrent instances;
   may require raising limits or lowering `max_instances` default.
4. Whether L4-only proxy is sufficient for all client libraries' reconnection and
   `startup`/SSL negotiation behavior (should be, since it's opaque bytes) — verify
   in issue 25 E2E.
5. Interaction of APFS local snapshots / Time Machine with large branch churn.
6. `pg_ctl` shutdown latency under load affecting switch/idle timing.

These are tracked in [`ISSUE_PLAN.md`](ISSUE_PLAN.md) §"Known unknowns".
