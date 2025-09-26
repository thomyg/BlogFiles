Below is a complete, zero-downtime migration plan to move a Postgres-backed SaaS from single-tenant (one DB per customer) to a multi-tenant cluster. It includes the chosen target design (row-based multi-tenancy using `tenant_id` with RLS), an alternative (schema-per-tenant) and where each fits, connection pooling, data migration choreography (backfill + CDC), cutover, rollback, validation, and ops playbooks—without asking you anything.

---

# 1) Target Architecture & Strategy

### Recommended strategy

* **Primary model:** **Row-based multi-tenancy** (shared tables) with a required `tenant_id` column on every tenant-scoped table, composite PKs/unique indexes including `tenant_id`, and Row-Level Security (RLS).
* **Why:** Highest density & lowest operational cost; simplest cross-tenant operational tooling; enables partitioning (by tenant hash or list) at high scale; single schema for app code.
* **When to consider schema-per-tenant (secondary path):** Rarely, for a handful of very large/regulatory tenants (isolation, custom extensions). Can coexist in the same cluster as an exception.

### ASCII architecture diagram

```
                       ┌────────────────────────────────────────┐
                       │               SaaS App                 │
                       │  - Tenant router middleware            │
  ┌────────────┐       │  - Adds tenant_id to each request      │
  │ End Users  │──────▶│  - Starts TX with SET LOCAL tenant_id  │
  └────────────┘       └───────────────┬────────────────────────┘
                                       │
                                 (TLS) │
                                       ▼
                                ┌───────────┐
                                │ PgBouncer │   transaction pooling
                                │  (HA)     │   auth: SCRAM, tls
                                └─────┬─────┘
                                      │
                         ┌────────────┴──────────────┐
                         ▼                           ▼
                ┌─────────────────┐         ┌─────────────────┐
                │  MT Primary     │◀───────▶│  MT Hot Standby │
                │  PostgreSQL     │  sync    │  PostgreSQL     │
                │  (RLS, Part.)   │  repl.   │  read scaling   │
                └────────┬────────┘         └─────────────────┘
                         │
                         │ writes
                         │
                         ▼
               ┌───────────────────────┐
               │  Kafka / Redpanda    │  (Change Data Capture bus)
               │  (Debezium topics)   │
               └─────────┬────────────┘
                         │
        ┌────────────────┴────────────────┐
        ▼                                 ▼
┌─────────────────────┐          ┌──────────────────────┐
│ Debezium Connectors │          │  JDBC Sink Connect   │
│ from each ST DB     │─────────▶│ upsert into MT tables│
│ (adds tenant_id SMT)│          │ (pk: tenant_id,id)   │
└──────────┬──────────┘          └──────────────────────┘
           │ CDC
           │
           ▼
┌──────────────────────────┐
│ Single-Tenant DBs (ST)   │  one per customer (current state)
│ pg_hba limited; logical  │
│ replication on; fdw used │  for bulk backfill
└──────────────────────────┘

Observability: Prometheus + Grafana (DB, PgBouncer, Kafka lag), Sentry/ELK for app; per-tenant SLOs.
```

### Key design choices (row-based)

* **`tenant_id`**: `uuid` (or ULID/UUIDv7). All tenant-scoped tables: composite primary key `(tenant_id, id)`; all uniques include `tenant_id`.
* **RLS**: Enabled per table; app sets `app.tenant_id` GUC at TX start; policies enforce `tenant_id = current_setting('app.tenant_id')::uuid`.
* **Partitioning**: Start with unpartitioned or **hash partition on `tenant_id`** into 8–32 partitions; for very large tenants, add **list partitions**.
* **Sequences/IDs**: Keep per-table `id` values as they are (ints or uuids), but all uniqueness is `(tenant_id, id)`; no global re-key required.
* **Connection pooling**: PgBouncer **transaction pooling**, disable server-side prepared statements in app driver, or use PgBouncer 1.21+ `statement_cache_mode=transaction`.

---

# 2) Schema Strategy Options

### A) Row-based with `tenant_id` (RECOMMENDED)

* **Pros:** Highest density; one schema; cross-tenant ops; strong RLS; simple versioning.
* **Cons:** Requires careful indexing; RLS foot-guns if GUC not set; query patterns must include `tenant_id`.

**Core DDL pattern**

```sql
-- 0. Shared helper: current tenant
CREATE OR REPLACE FUNCTION app.current_tenant() RETURNS uuid LANGUAGE sql STABLE
AS $$ SELECT current_setting('app.tenant_id', true)::uuid $$;

-- 1. Example table rewrite
ALTER TABLE public.invoice
  ADD COLUMN tenant_id uuid NOT NULL,
  ALTER COLUMN id SET NOT NULL;

-- Composite PK (preserves existing id uniqueness per tenant)
ALTER TABLE public.invoice
  DROP CONSTRAINT IF EXISTS invoice_pkey,
  ADD PRIMARY KEY (tenant_id, id);

-- Uniques must include tenant_id
CREATE UNIQUE INDEX IF NOT EXISTS ux_invoice_number_per_tenant
  ON public.invoice (tenant_id, invoice_number);

-- RLS
ALTER TABLE public.invoice ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON public.invoice
  USING (tenant_id = app.current_tenant());

-- Optional: Hash partitioning
-- CREATE TABLE public.invoice PARTITION BY HASH (tenant_id);
```

**App contract per TX**

```sql
-- At transaction start (middleware):
SET LOCAL app.tenant_id = '00000000-0000-0000-0000-000000000000';
-- All app queries MUST include tenant_id predicates or rely on RLS.
```

### B) Schema-per-tenant (niche)

* **Use only** for 1–5 gigantic / regulated tenants with divergent needs.
* **Operate** in parallel: place those tenants in dedicated schemas or DBs; app router maps them specially. All others use shared row-based model.

---

# 3) Connection Pooling (PgBouncer)

**`pgbouncer.ini` (minimal secure template)**

```
[databases]
appdb = host=mt-primary.internal port=5432 dbname=appdb auth_user=pgbouncer

[pgbouncer]
listen_port = 6432
listen_addr = 0.0.0.0
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
server_tls_sslmode = verify-full
server_tls_ca_file = /etc/ssl/certs/ca.pem
server_reset_query = DISCARD ALL
ignore_startup_parameters = extra_float_digits
pool_mode = transaction
max_client_conn = 5000
default_pool_size = 100
min_pool_size = 20
reserve_pool_size = 20
reserve_pool_timeout = 5
tcp_keepalive = 1
query_wait_timeout = 600
```

**Driver notes**

* Disable server-prepared statements (e.g., `prepareThreshold=0` for JDBC, or use PgBouncer statement caching only).
* Each request/transaction sets `SET LOCAL app.tenant_id = '<uuid>'`.

---

# 4) Data Migration Choreography (Zero-Downtime)

We’ll migrate **tenant-by-tenant**, without global downtime:

**Pillars**

1. **Backfill** from each Single-Tenant (ST) DB into Multi-Tenant (MT) tables using `postgres_fdw` (fast bulk copy).
2. **Change Data Capture (CDC)** streaming of ongoing ST writes via **Debezium → Kafka → JDBC sink** that upserts into MT tables with `(tenant_id, id)` PK.
3. **Per-tenant cutover** by updating the app’s **tenant router** to point that tenant to MT; keep CDC for grace period for rollback; then decommission ST.

### 4.1 Prepare MT cluster

**Global Postgres settings (MT cluster)**

```
wal_level = logical
max_replication_slots = 1024
max_wal_senders = 128
shared_buffers = 25% RAM
work_mem = 64MB
max_connections = 400           # app via PgBouncer
```

**Tenant registry**

```sql
CREATE TABLE app.tenants (
  tenant_id uuid PRIMARY KEY,
  external_key text UNIQUE NOT NULL, -- e.g., tenant slug
  source_dsn text NOT NULL,          -- ST DB conn string (for fdw/backfill)
  status text NOT NULL DEFAULT 'staging'  -- staging|live|decom
);
```

**RLS + composite keys across all tables**

* Repeat DDL pattern above for each tenant-scoped table.
* Add required indexes: for each frequent lookup, ensure first column is `tenant_id`.

### 4.2 For each tenant T (automatable), do:

> Shell env (example):

```
export TENANT_ID='f7332a0e-0f7d-4f22-978d-21b0a6021c8f'
export SRC_HOST='st-tenant-42.internal'
export SRC_DB='tenant42'
export SRC_USER='repl'
export SRC_PASS='*****'
export MT_HOST='mt-primary.internal'
export MT_DB='appdb'
export MT_USER='migrator'
export MT_PASS='*****'
```

#### (A) Backfill using `postgres_fdw`

On **MT primary**:

```sql
-- 1) Enable FDW
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

-- 2) Foreign server pointing to this tenant's ST DB
DROP SERVER IF EXISTS st_${TENANT_ID} CASCADE;
CREATE SERVER st_${TENANT_ID}
  FOREIGN DATA WRAPPER postgres_fdw
  OPTIONS (host '${SRC_HOST}', dbname '${SRC_DB}', port '5432');

-- 3) User mapping (use a role with SELECT only)
CREATE USER MAPPING FOR migrator
  SERVER st_${TENANT_ID}
  OPTIONS (user '${SRC_USER}', password '${SRC_PASS}');

-- 4) Import only whitelisted tables (repeat or scripted)
IMPORT FOREIGN SCHEMA public
  LIMIT TO (account, invoice, invoice_item, payment, user_account)
  FROM SERVER st_${TENANT_ID}
  INTO staging_${TENANT_ID};

-- 5) Speed up bulk load
SET maintenance_work_mem='2GB';
SET synchronous_commit='off';
SET work_mem='256MB';
```

Backfill **table-by-table** (idempotent, resumable). Example for `invoice`:

```sql
-- Ensure target columns match; add missing columns as NULL/defaults
-- Use INSERT ... SELECT with constant tenant_id
INSERT INTO public.invoice (tenant_id, id, customer_id, number, amount_cents, status, created_at, updated_at)
SELECT '${TENANT_ID}'::uuid, id, customer_id, number, amount_cents, status, created_at, updated_at
FROM staging_${TENANT_ID}.invoice
ON CONFLICT (tenant_id, id) DO NOTHING; -- allows retry

ANALYZE public.invoice;
```

Repeat for all tenant-scoped tables in dependency order (parents → children). For very large tables, chunk:

```sql
-- Chunked copy by primary key ranges
DO $$
DECLARE
  chunk_start bigint := 0;
  chunk_size  bigint := 100000;
  max_id      bigint;
BEGIN
  SELECT max(id) INTO max_id FROM staging_${TENANT_ID}.invoice;
  WHILE chunk_start <= max_id LOOP
    INSERT INTO public.invoice (tenant_id, id, customer_id, number, amount_cents, status, created_at, updated_at)
    SELECT '${TENANT_ID}'::uuid, id, customer_id, number, amount_cents, status, created_at, updated_at
    FROM staging_${TENANT_ID}.invoice
    WHERE id >= chunk_start AND id < chunk_start + chunk_size
    ON CONFLICT (tenant_id, id) DO NOTHING;

    chunk_start := chunk_start + chunk_size;
  END LOOP;
END$$;
```

#### (B) Start CDC for this tenant (Debezium)

**On ST DB (source), ensure logical replication user & privileges**

```sql
CREATE ROLE repl LOGIN REPLICATION PASSWORD '${SRC_PASS}';
ALTER SYSTEM SET wal_level = logical;
ALTER SYSTEM SET max_replication_slots = 16;
SELECT pg_reload_conf();
```

Ensure tables have **REPLICA IDENTITY** (UPSERT friendliness):

```sql
ALTER TABLE public.invoice REPLICA IDENTITY FULL;           -- if no natural PK
-- Preferably: ensure each table has a primary key for stable CDC
```

**Debezium Postgres connector (Kafka Connect REST)**

```bash
# POST to Kafka Connect
curl -s -X PUT http://connect:8083/connectors/debezium-${TENANT_ID}/config -H 'Content-Type: application/json' -d '{
  "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
  "database.hostname": "'${SRC_HOST}'",
  "database.port": "5432",
  "database.user": "repl",
  "database.password": "'${SRC_PASS}'",
  "database.dbname": "'${SRC_DB}'",
  "plugin.name": "pgoutput",
  "slot.name": "debezium_'${TENANT_ID}'",
  "publication.autocreate.mode": "filtered",
  "tombstones.on.delete": "false",
  "decimal.handling.mode": "double",
  "time.precision.mode": "adaptive_time_microseconds",
  "schema.include.list": "public",
  "table.include.list": "public.account,public.invoice,public.invoice_item,public.payment,public.user_account",
  "tombstone.handling.mode": "none",
  "snapshot.mode": "never",   // backfill already done by fdw
  "key.converter":"org.apache.kafka.connect.storage.StringConverter",
  "value.converter":"org.apache.kafka.connect.json.JsonConverter",
  "transforms": "AddTenant,unwrap",
  "transforms.AddTenant.type": "org.apache.kafka.connect.transforms.InsertField$Value",
  "transforms.AddTenant.static.field": "tenant_id",
  "transforms.AddTenant.static.value": "'${TENANT_ID}'",
  "transforms.unwrap.type":"io.debezium.transforms.ExtractNewRecordState",
  "transforms.unwrap.drop.tombstones":"true",
  "heartbeat.interval.ms":"5000"
}'
```

**JDBC sink to MT (upsert)**

```bash
curl -s -X PUT http://connect:8083/connectors/jdbc-sink-${TENANT_ID}/config -H 'Content-Type: application/json' -d '{
  "connector.class":"io.confluent.connect.jdbc.JdbcSinkConnector",
  "tasks.max":"4",
  "topics":"public.account,public.invoice,public.invoice_item,public.payment,public.user_account",
  "connection.url":"jdbc:postgresql://'${MT_HOST}':5432/'${MT_DB}'?sslmode=require",
  "connection.user":"'${MT_USER}'",
  "connection.password":"'${MT_PASS}'",
  "auto.create":"false",
  "auto.evolve":"false",
  "insert.mode":"upsert",
  "pk.mode":"record_value",
  "pk.fields":"tenant_id,id",
  "table.name.format":"public.${topic}",       // assumes same table names
  "delete.enabled":"true",
  "batch.size":"3000",
  "max.retries":"12",
  "retry.backoff.ms":"10000"
}'
```

**Result:** Ongoing changes on ST for tenant T stream to MT and are upserted with `tenant_id = TENANT_ID`.

#### (C) Catch-up & pre-cutover checks

* Wait until Kafka consumer lag for T’s topics is **0** and **no backfill gaps** exist.
* Lock out schema changes on ST for T (or route DDL via the migration channel).
* Run **validation script** (see §6) to compare counts & checksums.

#### (D) Per-tenant cutover (no downtime)

1. **Enable dual-write** in the app for T (temporary): app writes to both ST(T) and MT(T) for ~5–15 minutes. (Idempotent UPSERT on MT).
2. When CDC lag is 0 and dual-write is clean, **switch tenant router** so reads/writes go to MT for T.
3. Keep Debezium CDC running for **grace period** (e.g., 24–72h) for fast rollback.
4. Mark T as `status='live'` in `app.tenants`.

#### (E) Decommission tenant’s ST DB

* After grace period, stop dual-write; stop Debezium connectors for T; snapshot & archive DB; lockdown to read-only; eventually destroy.

---

# 5) Cutover Plan (Global Timeline)

1. **Week -2 to -1**: MT schema retrofit (add `tenant_id`, RLS, indexes); deploy PgBouncer; deploy router middleware; create tenants registry.
2. **Week -1**: Stand up Kafka + Connect + Debezium; smoke test with a sandbox tenant.
3. **Rolling (daily cadence)**:

   * For 10–50 tenants/day: Backfill (FDW), start CDC, validate, dual-write, **cutover** per tenant, monitor, keep CDC.
4. **Final**: After all tenants live on MT, turn off old infra.

---

# 6) Data Validation Tactics & Commands

**A. Row counts per table**

```sql
-- On ST (per tenant)
SELECT 'invoice' AS tbl, count(*) FROM public.invoice
UNION ALL SELECT 'invoice_item', count(*) FROM public.invoice_item
UNION ALL SELECT 'payment', count(*) FROM public.payment
ORDER BY 1;

-- On MT (scoped to tenant)
WITH t AS (SELECT '${TENANT_ID}'::uuid AS tid)
SELECT 'invoice' AS tbl, count(*) FROM public.invoice i, t WHERE i.tenant_id=t.tid
UNION ALL SELECT 'invoice_item', count(*) FROM public.invoice_item ii, t WHERE ii.tenant_id=t.tid
UNION ALL SELECT 'payment', count(*) FROM public.payment p, t WHERE p.tenant_id=t.tid
ORDER BY 1;
```

**B. Rolling checksums (memory-safe)**

```sql
-- ST
SELECT mod(hashagg, 1000000007) AS checksum
FROM (
  SELECT (hashtext(coalesce(id::text,''))::bigint
        + hashtext(coalesce(number,''))::bigint
        + hashtext(coalesce(status,''))::bigint
        + hashtext(coalesce(amount_cents::text,''))::bigint) AS hashagg
  FROM public.invoice
) s;

-- MT (tenant-scoped)
WITH t AS (SELECT '${TENANT_ID}'::uuid AS tid)
SELECT mod(hashagg, 1000000007) AS checksum
FROM (
  SELECT (hashtext(coalesce(id::text,''))::bigint
        + hashtext(coalesce(number,''))::bigint
        + hashtext(coalesce(status,''))::bigint
        + hashtext(coalesce(amount_cents::text,''))::bigint) AS hashagg
  FROM public.invoice i, t WHERE i.tenant_id=t.tid
) s;
```

**C. Spot-sample parity**

```sql
-- Take 100 random ids on ST, ensure existence & equality on MT:
WITH sample AS (
  SELECT id FROM public.invoice TABLESAMPLE SYSTEM (1) LIMIT 100
)
SELECT s.id,
       st.number AS st_number, mt.number AS mt_number,
       st.amount_cents AS st_amt, mt.amount_cents AS mt_amt
FROM sample s
JOIN public.invoice st ON st.id = s.id
LEFT JOIN public.invoice mt ON mt.tenant_id='${TENANT_ID}'::uuid AND mt.id=s.id
WHERE st.number IS DISTINCT FROM mt.number
   OR st.amount_cents IS DISTINCT FROM mt.amount_cents;
-- Expect 0 rows.
```

**D. CDC lag checks (Kafka)**

* Alert if per-topic **consumer lag > 0** for > 60s during cutover window.
* Alert if **throughput drops** or **error rate** increases.

**E. App end-to-end probes**

* Synthetic probe for T creating a tiny object and verifying it is readable in MT only after router switch.

---

# 7) Failure Playbook

**1) Backfill fails (FDW errors / DDL drift)**

* Action: Stop; fix schema mismatch; re-run the same `INSERT ... ON CONFLICT DO NOTHING` (idempotent).
* If specific rows fail: dump offending keys, correct data, re-insert.

**2) CDC lag grows (source write burst)**

* Action: Increase Kafka Connect tasks for sink (`tasks.max`), raise `batch.size`, provision MT indexes (ensure write-optimized indexes), scale I/O on MT, throttle noisy background jobs on ST.

**3) Conflicting unique constraints on MT**

* Cause: A cross-tenant unique missed `tenant_id`.
* Action: Drop/recreate the unique index to include `tenant_id`. Temporarily disable sink for that table, fix DDL, reprocess topic (Kafka replay).

**4) RLS blocking app traffic**

* Cause: `app.tenant_id` not set in a transaction.
* Action: Roll back the app deployment; add middleware guard that refuses queries if `app.tenant_id` missing; add DB default policy to **deny** rows when setting is NULL.

**5) PgBouncer saturation (pool exhaustion)**

* Action: Increase `default_pool_size`, add `reserve_pool_size`; verify app uses short transactions; eliminate server-prepared statements; scale PgBouncer horizontally behind a VIP.

**6) Cutover produces errors**

* Action: Immediately toggle tenant router back to ST (rollback); dual-write keeps MT up-to-date; investigate and retry later.

**7) Kafka/Connect outage during cutover**

* Action: Pause cutovers. For tenants mid-cutover, keep dual-write until CDC restored; no data loss (app writes ensure MT consistency).

**8) Rollback after cutover (within grace period)**

* You left CDC ST→MT on; to roll back, just flip router back to ST; MT already consistent; continue CDC while investigating. If you had stopped dual-write, use Kafka retained topics to rebuild any delta to ST via a **reverse sink** (MT→ST) configured analogously (temporary).

---

# 8) Rollback Plan (Per-Tenant)

**Window:** Grace period 24–72h after cutover; CDC remains enabled.

**Steps:**

1. **Flip router** for tenant T back to ST immediately.
2. Keep CDC ST→MT running; MT will continue to mirror ST.
3. Diagnose & fix (schema/index/app).
4. Re-validation; re-attempt cutover.
5. If CDC had been stopped: start **reverse sink** MT→ST to backfill MT-only writes:

   * Create a Debezium connector on MT (or logical decoding) and a JDBC sink to ST with pk `(id)` (ST schema), filtered by `tenant_id = T`.
6. Once ST reflects missed writes, proceed as in §4.2(D).

---

# 9) Alternative (Schema-Per-Tenant) Runbook (Brief)

If you must keep a few tenants isolated:

* Create `schema t_<TENANT_ID>`; copy data there; no RLS needed.
* App router maps those tenants to their schema (via `SET search_path`).
* Migration largely identical: FDW backfill + CDC into that schema’s tables.
* Cost: more DDL fan-out; less density. Prefer row-based for the majority.

---

# 10) Step-by-Step Operator Runbook (Copy-Paste Friendly)

**Prereqs (once)**

```bash
# create MT roles (run as superuser)
psql "sslmode=require host=$MT_HOST dbname=$MT_DB user=postgres" <<'SQL'
CREATE ROLE app LOGIN PASSWORD '***';
CREATE ROLE migrator LOGIN PASSWORD '***';
GRANT CONNECT ON DATABASE appdb TO app, migrator;
GRANT USAGE ON SCHEMA public TO app, migrator;
GRANT SELECT,INSERT,UPDATE,DELETE ON ALL TABLES IN SCHEMA public TO app, migrator;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT,INSERT,UPDATE,DELETE ON TABLES TO app, migrator;
SQL
```

**Per-tenant automation skeleton**

```bash
#!/usr/bin/env bash
set -euo pipefail

TENANT_ID="$1"
SRC_HOST="$2"; SRC_DB="$3"; SRC_USER="repl"; SRC_PASS="$4"
MT_HOST="mt-primary.internal"; MT_DB="appdb"; MT_USER="migrator"; MT_PASS="***"

export PGPASSWORD="$MT_PASS"

psql "host=$MT_HOST dbname=$MT_DB user=$MT_USER sslmode=require" <<SQL
-- registry
INSERT INTO app.tenants(tenant_id, external_key, source_dsn, status)
VALUES ('$TENANT_ID','${SRC_DB}','host=$SRC_HOST dbname=$SRC_DB user=$SRC_USER','staging')
ON CONFLICT (tenant_id) DO UPDATE SET source_dsn=excluded.source_dsn;

-- fdw server
CREATE EXTENSION IF NOT EXISTS postgres_fdw;
DROP SERVER IF EXISTS st_${TENANT_ID} CASCADE;
CREATE SERVER st_${TENANT_ID} FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host '$SRC_HOST', dbname '$SRC_DB', port '5432');
CREATE USER MAPPING IF NOT EXISTS FOR migrator SERVER st_${TENANT_ID}
OPTIONS (user '$SRC_USER', password '$SRC_PASS');

-- staging schema
DROP SCHEMA IF EXISTS staging_${TENANT_ID} CASCADE;
CREATE SCHEMA staging_${TENANT_ID};
IMPORT FOREIGN SCHEMA public LIMIT TO (account,invoice,invoice_item,payment,user_account)
FROM SERVER st_${TENANT_ID} INTO staging_${TENANT_ID};

-- bulk copy (repeat for each table in order)
INSERT INTO public.account (tenant_id,id,name,created_at,updated_at)
SELECT '$TENANT_ID'::uuid, id, name, created_at, updated_at
FROM staging_${TENANT_ID}.account
ON CONFLICT (tenant_id,id) DO NOTHING;

INSERT INTO public.user_account (tenant_id,id,email,role,created_at,updated_at)
SELECT '$TENANT_ID'::uuid, id, email, role, created_at, updated_at
FROM staging_${TENANT_ID}.user_account
ON CONFLICT (tenant_id,id) DO NOTHING;

INSERT INTO public.invoice (tenant_id,id,customer_id,number,amount_cents,status,created_at,updated_at)
SELECT '$TENANT_ID'::uuid, id, customer_id, number, amount_cents, status, created_at, updated_at
FROM staging_${TENANT_ID}.invoice
ON CONFLICT (tenant_id,id) DO NOTHING;

INSERT INTO public.invoice_item (tenant_id,id,invoice_id,sku,qty,price_cents)
SELECT '$TENANT_ID'::uuid, id, invoice_id, sku, qty, price_cents
FROM staging_${TENANT_ID}.invoice_item
ON CONFLICT (tenant_id,id) DO NOTHING;

INSERT INTO public.payment (tenant_id,id,invoice_id,amount_cents,method,paid_at)
SELECT '$TENANT_ID'::uuid, id, invoice_id, amount_cents, method, paid_at
FROM staging_${TENANT_ID}.payment
ON CONFLICT (tenant_id,id) DO NOTHING;

ANALYZE;
SQL

# Configure Debezium connectors (as shown earlier)
# ... curl PUT /connectors/debezium-$TENANT_ID/config ...
# ... curl PUT /connectors/jdbc-sink-$TENANT_ID/config  ...

echo "Tenant $TENANT_ID backfilled and CDC configured."
```

**Per-tenant cutover**

```bash
# Preconditions: consumer lag == 0; validations green.

# 1) Enable app dual-write for TENANT_ID (feature flag via your config store)
# 2) Wait 5–15 minutes; verify no errors in MT writes; CDC lag ~0.
# 3) Flip router for TENANT_ID to MT database (atomic config change).
# 4) Keep CDC running for grace period; stop dual-write after 24–72h if stable.
```

**Decommission**

```bash
# Stop connectors
curl -s -X DELETE http://connect:8083/connectors/debezium-${TENANT_ID}
curl -s -X DELETE http://connect:8083/connectors/jdbc-sink-${TENANT_ID}

# Lockdown ST DB to read-only; snapshot and archive (retain X days).
psql "host=$SRC_HOST dbname=$SRC_DB user=postgres" -c "ALTER SYSTEM SET default_transaction_read_only = on; SELECT pg_reload_conf();"
```

---

# 11) App Changes Checklist (Minimal)

* **Tenant router middleware**:

  * Extract tenant from auth (JWT claim, subdomain, header).
  * Start transaction; `SET LOCAL app.tenant_id = '<uuid>'`; enforce presence.
* **SQL**:

  * All queries either rely on RLS or include `WHERE tenant_id = :tenant`.
  * All inserts set `tenant_id` (via app default param or DB default from GUC).
* **Background jobs**:

  * Carry the tenant context through job payloads and `SET LOCAL` at start.

---

# 12) Performance & Indexing

* Add **leading `tenant_id`** in all major indexes used by OLTP paths.
* Consider **partial indexes** per frequent statuses (e.g., `(tenant_id,status)`).
* Partition large tables by **HASH(tenant_id)**; set `parallel_workers` for bulk ops.
* Routine **VACUUM (ANALYZE)**; track bloat; autovacuum tuned for write rates.

---

# 13) Security & Compliance

* **RLS mandatory**; default-deny policies; migrations run as `migrator` role.
* **PgBouncer TLS**, **SCRAM** auth.
* **Audit**: Log `app.tenant_id` (prefix `application_name`) for all connections.
* **Secrets** in vault; **pg_hba** restricted to infra CIDRs.

---

# 14) Final Self-Check List (Go/No-Go)

**Schema & App**

* [ ] All tenant tables have `tenant_id uuid NOT NULL`.
* [ ] All PKs/uniques include `(tenant_id, ...)`.
* [ ] RLS enabled with `tenant_id = app.current_tenant()`.
* [ ] App sets `SET LOCAL app.tenant_id` at TX start; guardrails in middleware.
* [ ] Prepared statements disabled (or PgBouncer session pooling not used).

**Infra**

* [ ] PgBouncer deployed (HA), `pool_mode=transaction`, TLS on.
* [ ] MT cluster wal/logical settings applied; replicas healthy.
* [ ] Kafka + Connect + Debezium up; monitoring on lag & error topics.

**Migration**

* [ ] FDW backfill scripts tested on a staging tenant.
* [ ] Debezium snapshot disabled (we backfill), topics live.
* [ ] JDBC sink upsert with `(tenant_id,id)` configured; deletes enabled.
* [ ] Validation queries scripted; thresholds defined (0 deltas except known).

**Cutover**

* [ ] Dual-write feature flag implemented & tested.
* [ ] Router switch is atomic, auditable, and reversible per tenant.
* [ ] Rollback steps documented; CDC kept for grace period.

**Observability**

* [ ] Dashboards: MT CPU/IO, locks, bloat, slow queries; PgBouncer pools; Kafka lag.
* [ ] Alerts: RLS violations (0 expected), sink errors, rising lag, pool saturation.

**Data safety**

* [ ] Point-in-time recovery (PITR) for MT enabled.
* [ ] ST DB snapshots taken before decommission.
* [ ] Kafka retention ≥ grace period + safety buffer.

If all boxes are checked and a pilot tenant completed cleanly → **GO**.

---

## Appendix: Notes on Limits & Variants

* **If Kafka/Debezium is not desired**: use **pglogical** or Postgres 15+ **logical replication with row filtering** into **per-tenant staging tables**, then `INSERT ... ON CONFLICT` into MT (adds operational complexity but removes Kafka).
* **Hot path latency**: Keep PgBouncer near DB; use prepared-statement alternatives compatible with transaction pooling.
* **Very large tenants**: Promote to **list partitions** (one partition per huge tenant); optional separate tablespace/IO class.

---

This plan lets you migrate tenants continuously, without downtime, with per-tenant cutover and clean rollback, while landing on a secure, scalable multi-tenant design.
