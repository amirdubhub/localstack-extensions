# Brancher-backed LocalStack RDS snapshots / clones

**Status:** design only (no implementation in this PR)  
**Branch intent:** propose phase-1 architecture for wiring [Baseshift Brancher](https://baseshift.com) into LocalStack RDS snapshot/restore(+ eventual CoW clone) verbs, starting from the existing Baseshift extension.  
**Audience:** Baseshift (Amir) + LocalStack (Waldemar / RDS maintainers)

---

## 1. Goals / non-goals

### Goals (phase 1 — pure Brancher)

1. Give LocalStack RDS **PostgreSQL / aurora-postgresql** (and later MySQL if Brancher-aligned) **fast, CoW-friendly snapshot / restore / clone** semantics by driving **Brancher** under the hood.
2. Map familiar AWS RDS verbs onto Brancher ops so app/test code can keep using `CreateDBSnapshot`, `RestoreDBInstanceFromDBSnapshot`, `Describe*`, `Delete*`, and (later) Aurora-style CoW clone via `RestoreDBClusterToPointInTime` + `RestoreType=copy-on-write`.
3. Ship as an **opt-in mode** of this extension (or a tightly scoped companion mode), without breaking today’s Docker-image (Dub) clone path.
4. Reuse existing extension strengths: gateway Postgres TCP routing on `:4566`, host-port allocation, clones lifecycle bookkeeping.
5. Keep phase 1 **self-contained**: Brancher root + engine binaries managed beside LocalStack; **no Dub ingest** required to prove the loop.

### Non-goals (phase 1)

- Implementing Dub → Brancher ingest / masked Docker snapshot conversion (phase 2 only; see §8).
- Full AWS parity for `CopyDBSnapshot*`, cross-region copy, restore-from-S3, automated backup retention windows, or PITR replay of WAL.
- Replacing LocalStack Cloud Pods / persistence with Brancher (orthogonal mechanisms).
- MySQL/MariaDB/MSSQL RDS snapshot parity on engines LocalStack does not snapshot today (Brancher can do MySQL/Mongo locally, but LS RDS API surface for those engines is the limiting factor — call out as follow-up).
- EULA / licensing policy work (explicitly out of scope per product direction).
- Shipping production-ready Brancher packaging inside LocalStack core in this design PR.

---

## 2. Today: LocalStack RDS snaps/clones vs gap

### What LocalStack RDS already does (native provider since 4.4)

Documented / verified behavior relevant here:

| Area | Behavior today |
|------|----------------|
| Engines with real local DB | `postgres` / `aurora-postgresql` → real Postgres (major versions 13–17, dynamic install). MySQL → Docker; MariaDB/MSSQL → local/package or Docker. |
| Snapshots implemented | `Create` / `Delete` / `Describe` for DB + cluster snapshots; `RestoreDBInstanceFromDBSnapshot`; `RestoreDBClusterFromSnapshot`. |
| Snapshots **not** implemented | `CopyDBSnapshot*`; `Restore*ToPointInTime` (AWS Aurora CoW clones use `RestoreDBClusterToPointInTime` + `RestoreType=copy-on-write`); restore-from-S3. |
| Engine gaps | MySQL / MariaDB / MSSQL: RDS API snapshots unsupported. |
| Persistence | Cloud Pods / LS persistence ≠ RDS snapshots (and pre-4.4 pods are incompatible with the native provider). |

Practically: Postgres snapshot/restore works as a **full data copy / rehydrate** path suitable for correctness tests, but **not** as a cheap CoW branch/clone for agent/CI fan-out.

### What this Baseshift extension does today

Verified in `localstack_baseshift/extension.py` + README:

- Runs **Baseshift Docker snapshot (Dub) images** as sibling containers (`ls-baseshift-clone-*`).
- HTTP clones API at `baseshift.*.localstack.cloud:4566/clones` (start/list/get/stop).
- Postgres handshake routing on gateway `:4566` via `patch_gateway_for_tcp_routing` / `register_tcp_extension`; host ports `5432` then `15432+`.
- Env: `BASESHIFT_IMAGE`, `BASESHIFT_ENCRYPTION_PASSWORD`, `BASESHIFT_CLONE_*` passthrough, `BASESHIFT_DB_TYPE`.
- **No Brancher integration; no RDS API interception.**

### Gap this plan closes

| Need | Native LS RDS | Dub Docker clones (extension today) | Brancher (proposed) |
|------|---------------|--------------------------------------|---------------------|
| AWS RDS snapshot API | Yes (Postgres) | No | Yes (mapped) |
| Instant / CoW clone | No | No (container from image; full writable volume) | Yes (libc CoW) |
| Masked prod Dub | N/A | Yes | Phase 2 ingest |
| Survive reboot as live session | DB files via LS storage | Containers ephemeral unless recreated | Clone **sessions** do not survive reboot; snaps/refs on disk do |

Phase 1 targets the middle column → right column for **local RDS-shaped workflows**, without waiting on Dub ingest.

---

## 3. Proposed architecture (phase 1)

### Mental model

Treat each LocalStack RDS Postgres instance/cluster endpoint that opts into Brancher mode as a **Brancher-managed database**:

1. Extension (or Brancher helper) owns a **Brancher root** (e.g. `/var/lib/localstack/baseshift-brancher/` or host-mounted `~/.localstack/baseshift-brancher/`) containing `.brancher/` (`chunkpool/`, `snaps/`, `refs/`).
2. The live “primary” for an RDS resource is a Brancher **base** (after `brancher init` / `brancher db install` + first load), not an unmanaged `postgres` process.
3. RDS snapshot identifiers map to Brancher **snap/branch refs**.
4. RDS restore / CoW clone identifiers map to Brancher **clone sessions** (writable CoW checkouts) bound to host ports (+ optional gateway route).

```text
  awslocal rds CreateDBSnapshot
            │
            ▼
  ┌─────────────────────────────┐
  │ Baseshift extension         │
  │  BrancherRdsBridge (new)    │──► brancher branch|snap …
  └─────────────┬───────────────┘
                │ metadata: SnapshotIdentifier ↔ brancher ref
                ▼
         .brancher/ (chunkpool, snaps, refs)

  awslocal rds RestoreDBInstanceFromDBSnapshot
            │
            ▼
  BrancherRdsBridge ──► brancher clone <ref> ──► port N
            │
            ▼
  reuse gateway TCP router (Postgres handshake) when N is the “primary” clone port
```

### Brancher root lifecycle (inside / beside LocalStack)

| Lifecycle event | Behavior |
|-----------------|----------|
| Extension enable + `BASESHIFT_RDS_BACKEND=brancher` | Ensure Brancher CLI present (`doctor`); `init` root if missing; ensure `/dev/shm` usable (see §6). |
| `CreateDBInstance` / cluster becomes available (Postgres) | Either (A) create instance **already under Brancher** from empty/base template, or (B) **adopt** an existing LS Postgres data dir into Brancher via documented import/`init` path. Prefer (A) for phase 1 simplicity if hooks allow. |
| Snapshot create | Quiesce or use Brancher-consistent snap; record AWS-shaped metadata in extension store (SQLite/JSON under LocalStack data dir). |
| Restore / clone | `brancher clone` → allocate port → wait ready → register TCP route if Postgres default. |
| Delete snapshot | `brancher rm` ref + `gc` when safe. |
| Platform shutdown | `brancher stop` active clones; leave snaps/refs on disk; note: live clone sessions won’t resume across reboot without re-clone. |
| LS restart | Rehydrate Describe* from metadata; optionally auto re-clone “default” restored instances if marked durable. |

### Verb → Brancher mapping (phase 1)

| AWS / LocalStack RDS API | Brancher ops (proposed) | Notes |
|--------------------------|-------------------------|-------|
| `CreateDBSnapshot` / `CreateDBClusterSnapshot` | `brancher branch` or snap create from live session/base | SnapshotIdentifier → ref name (sanitize to Brancher naming). |
| `DescribeDBSnapshots` / `DescribeDBClusterSnapshots` | Metadata store + `brancher ls` / `stats` for size/status | Status `creating`→`available`. |
| `RestoreDBInstanceFromDBSnapshot` | `brancher clone <snap>` → new endpoint | New DBInstanceIdentifier; allocate port like today’s clones API. |
| `RestoreDBClusterFromSnapshot` | Same clone path; cluster+instance bookkeeping | Match LS Aurora pattern (restore cluster, then instance). |
| `DeleteDBSnapshot` / `DeleteDBClusterSnapshot` | `brancher rm` + opportunistic `gc` | Refuse if clones still depend on ref (or promote/squash policy). |
| `DeleteDBInstance` (restored clone) | `brancher stop` + remove session | Does not delete the snap unless requested. |
| *(later)* `RestoreDBClusterToPointInTime` + `RestoreType=copy-on-write` | Instant `brancher clone` from latest snap/base | Closest AWS CoW clone analogue; not in LS today — high product value. |
| `CopyDBSnapshot*` | Out of scope phase 1 | Could be ref alias / `brancher` copy within same root later. |

### Optional mode flag

Recommend a single explicit switch (names bikeshedable):

```bash
# opt into Brancher-backed RDS snapshot/restore for Postgres
LOCALSTACK_BASESHIFT_RDS_BACKEND=brancher

# keep today’s Dub Docker clones API unchanged
LOCALSTACK_BASESHIFT_IMAGE=...   # independent; can coexist if ports don’t collide
```

Semantics:

- `BASESHIFT_RDS_BACKEND=native` (default): no RDS interception; extension behaves as today (Dub Docker only).
- `BASESHIFT_RDS_BACKEND=brancher`: enable Brancher bridge for snapshot/restore(+ clone) verbs on supported engines.
- Dub clones API remains available unless we later add `BASESHIFT_CLONE_BACKEND=brancher` for the HTTP `/clones` surface (nice-to-have; not required for phase 1 RDS mapping).

---

## 4. Where to hook — options and recommendation

### Option A — Extension intercepts RDS APIs (recommended primary)

**Idea:** Keep Brancher ownership in `localstack-baseshift`. On `update_gateway_routes` / platform hooks, register handlers or service patches that wrap snapshot/restore operations when `BASESHIFT_RDS_BACKEND=brancher`.

**Pros**

- Iterates in this repo (Amir’s fork / extension) without blocking on LocalStack core merges for every experiment.
- Clear product packaging: “Baseshift extension adds Brancher CoW to local RDS.”
- Reuses existing port allocation + gateway TCP routing code paths.
- Opt-in flag keeps default LS RDS behavior intact.

**Cons / risks**

- Intercepting a subset of RDS can diverge from native provider state machines (ARNs, statuses, waiters) unless we carefully mirror responses.
- May need fragile patches into how the native provider starts Postgres (data dir, port, process model) to **adopt** instances under Brancher.
- Extensions are weaker than first-class provider backends for deep engine lifecycle.

**Mitigation:** phase 1 scopes only snapshot/restore/delete/describe for instances **created or tagged under Brancher mode**, and returns clear `InvalidParameter` / logged fallback for unsupported combinations.

### Option B — Patch / extend native LocalStack RDS provider

**Idea:** Upstream a Brancher storage backend inside LocalStack core RDS (Postgres path).

**Pros:** Cleanest long-term parity; waiters/state live in one place.  
**Cons:** Longer review cycle; Brancher binary/EULA packaging questions land in core; slows Baseshift-led iteration.  
**Use later:** once phase 1 proves the mapping in the extension, propose upstreaming the backend interface.

### Option C — Shadow clones API only (Brancher behind `/clones`)

**Idea:** Change or add `POST /clones` to start Brancher clones instead of Docker images; do **not** map RDS verbs yet.

**Pros:** Smallest change to current extension.  
**Cons:** Misses Amir’s product goal of RDS-shaped snaps/clones; apps using `awslocal rds` see no benefit.  
**Role:** useful **internal** primitive the RDS bridge calls, not the primary user surface.

### Recommendation

**Primary approach: Option A (extension-owned Brancher RDS bridge), with Option C as the internal clone runtime, and Option B as a phase-1.5/2 upstream candidate.**

Concrete shape:

1. New module (illustrative) `localstack_baseshift/brancher.py` — CLI wrapper (`init`, `branch`, `clone`, `stop`, `rm`, `gc`, `doctor`, `ls`, `stats`).
2. New `BrancherRdsBridge` — maps RDS snapshot/restore calls ↔ Brancher + metadata.
3. Optional thin `POST /clones` enhancement for Brancher-backed clones (same JSON shape, `backend: "brancher"`), so demos/tests can exercise CoW without full RDS.
4. Keep Docker Dub path unchanged behind existing code.

---

## 5. Engine binary / version alignment with LocalStack Postgres

LocalStack Postgres for RDS:

- Majors **13–17** (dynamic install); default **17**; minor not selectable; `RDS_PG_CUSTOM_VERSIONS=0` forces default 17.
- Describe APIs may still report the requested `EngineVersion` while the installed binary differs.

Brancher:

- Uses **libc interposition** around real `postgres` / `mysqld` / `mongod` binaries (`brancher db install`, etc.), not a Docker service model.
- Clone correctness requires the **same major engine** that wrote the snap chunks.

Phase 1 rules:

1. **Pin Brancher’s Postgres major to the LS instance’s effective major** (read from running server / LS install path, not only from `EngineVersion` string).
2. Prefer Brancher using the **same binary LocalStack already installed** for that major when possible (symlink / `PATH` / Brancher db install pointing at LS Postgres), to avoid catalog/version skew.
3. Document unsupported: restore snap taken on PG 15 into a Brancher clone forced to PG 17 (reject early with a clear error).
4. MySQL: Brancher supports MySQL, but LS RDS snapshots for MySQL are unimplemented — treat as **phase 1.b** only after Postgres path is green (and decide whether to expose via RDS API or only `/clones`).
5. CI smoke should assert `SELECT version()` major matches the snap’s recorded major.

Open alignment work with Waldemar: where LS stores Postgres installs inside the container, and whether those binaries are safe to `LD_PRELOAD`/interpose under Brancher.

---

## 6. `/dev/shm`, ports, and gateway TCP routing

### Shared memory

Brancher’s CoW chunkpool typically wants a large, fast shared-memory / scratch area. LocalStack runs in Docker:

- Ensure the LocalStack container (and any Brancher helper) is started with adequate `--shm-size` (e.g. `1g`+; size TBD by Brancher docs / `doctor`).
- Prefer Brancher root on a volume that survives container recreate if snaps should persist across `lstk` restarts; **do not** assume `/dev/shm` alone is durable.
- Extension startup should run `brancher doctor` and fail fast with actionable logs if shm/root permissions are wrong.

### Ports

Reuse the extension’s allocator patterns:

| Role | Port strategy |
|------|----------------|
| Brancher clone (Postgres) “primary” | Prefer `5432` if free — enables gateway routing like today’s first Dub clone. |
| Additional clones | `15432–15531` (existing `EXTRA_PORT_RANGE`) or a dedicated Brancher range if Dub clones coexist. |
| Native LS RDS instances | Today use dynamic ports (e.g. `4510+`). Brancher-adopted primaries may keep LS-assigned ports **or** be re-bound — decide in implementation; recommendation: **keep LS endpoint ports** for restored instances so `DescribeDBInstances` stays honest, and only use `5432`/`15432+` for extension `/clones` Brancher sessions. |

Collision rule when Dub + Brancher modes coexist: single port allocator shared across both backends.

### Gateway TCP routing

Reuse `is_postgres_handshake` + `register_tcp_extension` / `unregister_tcp_extension` from the current extension:

- Only one Postgres backend should own the gateway handshake route (already a documented limitation vs ParadeDB).
- Policy proposal: gateway route attaches to the **first running Postgres Brancher/Dub clone on the default host port**, same as today; RDS instance endpoints continue to use their dedicated ports (as LS does now).
- MySQL remains host-port only (server-first protocol).

---

## 7. Acceptance tests / smoke sequence

Phase 1 should land tests before broad marketing. Suggested layers:

### 7.1 Unit / contract (no full LS)

- CLI wrapper dry-run / mocked subprocess: ref name sanitization, verb mapping, error mapping.
- Metadata store: SnapshotIdentifier ↔ brancher ref round-trip.

### 7.2 Integration smoke (LocalStack + Brancher + extension)

```bash
# 0. prerequisites
brancher doctor
# LocalStack with extension, BASESHIFT_RDS_BACKEND=brancher, adequate shm

# 1. create Postgres instance
awslocal rds create-db-instance --db-instance-identifier src \
  --engine postgres --engine-version 17 \
  --master-username test --master-user-password test --db-instance-class db.t3.micro
awslocal rds wait db-instance-available --db-instance-identifier src

# 2. seed
psql "host=localhost port=<p> user=test dbname=test" -c "CREATE TABLE t(id int); INSERT INTO t VALUES (1);"

# 3. snapshot  (= brancher snap/branch)
awslocal rds create-db-snapshot --db-instance-identifier src --db-snapshot-identifier snap1
awslocal rds wait db-snapshot-available --db-snapshot-identifier snap1

# 4. mutate source (must not appear in restore)
psql ... -c "INSERT INTO t VALUES (2);"

# 5. restore (= brancher clone)
awslocal rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier clone1 --db-snapshot-identifier snap1
awslocal rds wait db-instance-available --db-instance-identifier clone1

# 6. assert CoW isolation + content
psql ...clone1... -c "SELECT * FROM t;"   # expect only row 1
psql ...clone1... -c "INSERT INTO t VALUES (3);"
psql ...src...    -c "SELECT * FROM t;"   # expect 1,2 — not 3

# 7. delete snapshot / instance cleanup
awslocal rds delete-db-instance --db-instance-identifier clone1 --skip-final-snapshot
awslocal rds delete-db-snapshot --db-snapshot-identifier snap1
# brancher ls/gc shows ref removed; doctor clean
```

### 7.3 Optional CoW latency assertion

- Time restore of a multi-GB seeded DB; expect Brancher clone startup ≪ native full restore (threshold TBD once numbers exist).
- `brancher stats` shows shared chunks across src snap + clone.

### 7.4 Regression

- With `BASESHIFT_RDS_BACKEND` unset: existing Dub Docker tests (`tests/test_extension.py`) still pass.
- Gateway Postgres handshake still works for Dub **or** Brancher primary, not both fighting ParadeDB.

---

## 8. Phase 2 sketch — Dub → Brancher ingest (brief)

Only after phase 1 RDS↔Brancher loop is solid:

1. **Input:** Baseshift Docker snapshot image (masked Dub) — same artifact the extension pulls today (`BASESHIFT_IMAGE` / ECR).
2. **Ingest:** One-shot job extracts/loads Dub data into a Brancher **base snap** (or `brancher init` from a running clone container’s data directory), then discards the heavy container.
3. **Serve:** Subsequent `CreateDBSnapshot` / clone / `/clones` fan-out are pure Brancher CoW from that base — fast PR/agent clones without re-pulling Docker layers each time.
4. **Control plane:** optional link from Baseshift Cloud “get docker snapshot” → local ingest → Brancher ref named after Dub/version.
5. **Non-goal for phase 2 design here:** re-implement masking; masking stays in Baseshift replication server / Dub build.

```text
Dub Docker image ──ingest──► Brancher base snap ──clone──► many local CoW DBs
                              ▲
                              └── phase 1 already understands snaps/clones via RDS verbs
```

---

## 9. Open questions for Waldemar / LocalStack

1. **Adoption hook:** Can an extension reliably wrap only snapshot/restore while CreateDBInstance stays in the native provider, or do we need a first-class “external engine supervisor” interface in RDS?
2. **Postgres binary reuse:** Absolute paths / install layout for majors 13–17 inside the LocalStack container; is interposition (Brancher) supported against those binaries?
3. **Data directory ownership:** Where does native RDS keep PGDATA, and is cold-import into Brancher acceptable, or must instances be Brancher-born?
4. **CoW API surface:** Preference for exposing Aurora-style `RestoreDBClusterToPointInTime` + `RestoreType=copy-on-write` in LocalStack (even as extension-emulated) vs inventing a non-AWS extension API?
5. **Port / endpoint contract:** Must restored instances keep `DescribeDBInstances` ports identical to native behavior when Brancher supervises the process?
6. **Persistence interaction:** Should Brancher roots live under LocalStack’s persistence volume? How should Cloud Pods treat Brancher chunkpools?
7. **`/dev/shm` defaults:** Will LocalStack document/raise default `shm-size` for Brancher-enabled installs?
8. **Upstream path:** If phase 1 works in the extension, is LocalStack open to a pluggable RDS storage/snapshot backend interface?
9. **MySQL:** Worth Brancher-backed `/clones` for MySQL before any RDS API work, given native MySQL snapshots are unsupported?
10. **Security / multi-tenant CI:** Brancher root permissions when LocalStack runs as non-root; multiple concurrent LS containers sharing a host chunkpool — supported or explicitly unsupported?

---

## Appendix A — Current extension anchors (for implementers)

| Piece | Location |
|-------|----------|
| Extension entry | `localstack_baseshift/extension.py` → `BaseshiftExtension` |
| Clones HTTP API | `ClonesApi` routes on `baseshift.<domain>` |
| Docker run path | `_run_clone` / `DOCKER_CLIENT.run_container` |
| Gateway Postgres routing | `is_postgres_handshake`, `register_tcp_extension` |
| Port allocation | `DB_PORTS`, `EXTRA_PORT_RANGE`, `_allocate_port` |
| Tests | `baseshift/tests/test_extension.py` (Dub/stand-in Postgres image) |

## Appendix B — Brancher command surface (phase 1 subset)

Expect to shell out to (names per Brancher CLI):  
`init`, `branch`, `clone`, `stop`, `promote`, `squash`, `stats`, `ls`, `rm`, `gc`, `doctor`, `db install`.

Phase 1 minimum viable: **init, doctor, branch/snap, clone, stop, ls, rm, gc, stats**.  
`promote` / `squash` deferred until we define “make clone the new primary” semantics for RDS modify/failover-like flows.

---

## Decision summary

| Topic | Decision for phase 1 |
|-------|----------------------|
| Backend | **Pure Brancher** (no Dub ingest) |
| User surface | **RDS snapshot/restore APIs** (Postgres first) |
| Hook location | **Baseshift extension bridge** (Option A), Brancher runtime shared with optional `/clones` |
| Coexistence | Opt-in `BASESHIFT_RDS_BACKEND=brancher`; Dub Docker path unchanged |
| CoW AWS verb | Plan for `RestoreDBClusterToPointInTime` + `copy-on-write` once basics work |
| Phase 2 | Ingest Dub Docker snapshot → Brancher base snap |
