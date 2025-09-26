Below is a **complete, zero‑downtime migration plan** to move a Postgres‑backed SaaS from **single‑tenant** (one DB per tenant or one DB for a single customer) to **multi‑tenant**. I’ve **decided the target design** is **row‑based multi‑tenancy** using a `tenant_id` column + **Row Level Security (RLS)**, hash‑partitioned by `tenant_id` for scale. I also include a **schema‑per‑tenant** path where that’s the better tradeoff (e.g., special isolation needs), and a **hybrid** option to mix both.

You get:

1) **ASCII architecture diagram**  
2) **Step‑by‑step runbook with precise commands** (Postgres SQL & shell)  
3) **Data validation tactics** (deterministic counts & checksums, integrity and RLS tests)  
4) **Failure playbook** (what breaks, how to see it, what to do—fast)  
5) **Final self‑check list** proving readiness

> **Assumptions (decided):**
> - Current: **multiple single‑tenant Postgres databases** (one per tenant). If it’s only one, treat N=1 and follow the same flow.  
> - Target: **one primary multi‑tenant Postgres database** with **row‑based** tenancy + **RLS**; **hash partitions** by `tenant_id` for scale; **pgBouncer** for pooling.  
> - Migration uses **postgres_fdw** for safe, pull‑based **backfill** + lightweight **trigger‑based CDC** (change data capture) on source DBs until cutover.  
> - **No downtime** for tenants during migration and cutover (you’ll perform tenant‑by‑tenant cutovers with canaries).  
> - Primary keys: existing numeric `id` remains but **composite PK** becomes `(tenant_id, id)`. New records can start using UUIDv7 (or keep numeric) as you prefer.



---

## 1) Architecture (ASCII)

```
                                 ┌─────────────────────────────────────────────┐
 Client Traffic                   │                 Application                │
  ───────────▶  HTTPS/API GW  ───▶│  App services (stateless)                 │
                                 │  - Sets tenant_id per request/txn          │
                                 │  - Dual-write logic (limited window)       │
                                 └───────────────┬────────────────────────────┘
                                                 │
                                      (transaction pooling)
                                                 │
                                         ┌───────▼────────┐
                                         │   PgBouncer    │
                                         │  (txn mode)    │
                                         └───────┬────────┘
                                                 │
                                 ┌───────────────▼───────────────────────┐
                                 │        Multi-tenant Postgres          │
                                 │  Database: app_prod (target)          │
                                 │  - RLS enforced on tenant_id          │
                                 │  - Hash partitioning on tenant_id     │
                                 │  - Outbox (for post-cutover rollback) │
                                 └──────┬─────────┬──────────┬───────────┘
                                        │         │          │
                               ┌────────▼───┐  ┌──▼────────┐ └────────────┐
            FDW (pull)         │fdw_t_101  │  │fdw_t_202  │  ... fdw_t_N │
             + CDC             │(foreign   │  │(foreign   │              │
     ┌───────────────────┐     │ schema)   │  │ schema)   │              │
     │  Migrator/Synchron│     └────┬──────┘  └────┬──────┘              │
     │   (backfill & CDC)│          │              │                      │
     └───────┬───────────┘   INSERT SELECT    INSERT SELECT               │
             │                                                      ┌─────▼───────┐
             │               ┌─────────────────────────────────────▶│ Monitoring  │
             │               │                                      │ & Alerting  │
             │               │   Changes via triggers               └─────────────┘
             │               │   to per-tenant Outbox tables
             │               │
   ┌─────────▼──────────┐    │     ┌─────────────────────────┐
   │ Source Tenant DB 1 │────┘     │ Source Tenant DB 2 ...  │─── ... Tenant DB N
   │ (single-tenant)    │          │ (single-tenant)         │
   │ - app tables       │          │ - app tables            │
   │ - outbox_* (CDC)   │          │ - outbox_* (CDC)        │
   └────────────────────┘          └─────────────────────────┘
```

**Cutover per tenant**: drain CDC lag to zero → switch app reads/writes to target → keep reverse‑CDC for a short safety window for rollback → finalize.



---

## 2) Schema Strategy

### Decision
- **Default**: **Row‑based** (single shared tables) with `tenant_id UUID` (or `BIGINT`) + RLS.  
  - Pros: lowest operational overhead, best pooling efficiency, easiest cross‑tenant analytics, good scale with partitioning.  
  - Cons: strongest need to **prove isolation**; must correctly implement RLS; careful with connection/session state under pooling.

- **Alternative**: **Schema‑per‑tenant** (`tenant_123.orders`), within the **same database**.  
  - Pros: stronger logical isolation; simpler per‑tenant maintenance.  
  - Cons: many schemas = metadata bloat, migration complexity, less pooling efficiency, code complexity.

- **Hybrid**: **Row‑based for most tenants**, **schema‑per‑tenant for VIP/regulatory** tenants. Your app routes accordingly.

### Row‑based design (chosen)
- **Tenant registry**
  ```sql
  create table if not exists tenants (
    tenant_id uuid primary key,
    slug      text unique not null,
    source_dsn text,             -- connection string to source (optional)
    state     text not null check (state in ('new','backfilling','cdc','cutover','postcutover','done','paused')),
    created_at timestamptz not null default now(),
    updated_at timestamptz not null default now()
  );

  create trigger tenants_updated_at
  before update on tenants
  for each row execute function pg_catalog.setval(''); -- placeholder if you use a generic updated_at trigger
  ```
- **Core tables** (example)
  ```sql
  -- Enable row security globally (belt & suspenders)
  alter database app_prod set row_security = on;

  -- Example: accounts, users, orders
  create table if not exists accounts (
    tenant_id uuid not null,
    id        bigint not null,
    name      text not null,
    created_at timestamptz not null default now(),
    updated_at timestamptz not null default now(),
    primary key (tenant_id, id)
  ) partition by hash (tenant_id);

  create table if not exists users (
    tenant_id uuid not null,
    id        bigint not null,
    account_id bigint not null,
    email     text not null,
    created_at timestamptz not null default now(),
    updated_at timestamptz not null default now(),
    primary key (tenant_id, id),
    unique (tenant_id, email),
    foreign key (tenant_id, account_id) references accounts(tenant_id, id)
  ) partition by hash (tenant_id);

  create table if not exists orders (
    tenant_id uuid not null,
    id        bigint not null,
    account_id bigint not null,
    total_cents bigint not null,
    status    text not null check (status in ('pending','paid','canceled','refunded')),
    created_at timestamptz not null default now(),
    updated_at timestamptz not null default now(),
    primary key (tenant_id, id),
    foreign key (tenant_id, account_id) references accounts(tenant_id, id)
  ) partition by hash (tenant_id);

  -- Create partitions (example: 32)
  do $$
  declare i int;
  begin
    for i in 0..31 loop
      execute format('create table if not exists accounts_p%1$s partition of accounts for values with (modulus 32, remainder %1$s);', i);
      execute format('create table if not exists users_p%1$s    partition of users    for values with (modulus 32, remainder %1$s);', i);
      execute format('create table if not exists orders_p%1$s   partition of orders   for values with (modulus 32, remainder %1$s);', i);
    end loop;
  end$$;
  ```

- **RLS policies** (require app to set `app.tenant_id` per transaction)
  ```sql
  -- Create a GUC namespace (no DDL required; we just use it)
  -- Application MUST set: set local app.tenant_id = '<uuid>';

  -- Secure default: block if app.tenant_id is not set
  create or replace function current_tenant() returns uuid
  language sql stable
  as $$ select nullif(current_setting('app.tenant_id', true), '')::uuid $$;

  -- RLS per table
  alter table accounts enable row level security;
  create policy tenant_isolation_accounts on accounts
    using (tenant_id = current_tenant());

  alter table users enable row level security;
  create policy tenant_isolation_users on users
    using (tenant_id = current_tenant());

  alter table orders enable row level security;
  create policy tenant_isolation_orders on orders
    using (tenant_id = current_tenant());
  ```

> **App requirement** (with PgBouncer transaction pooling):  
> - Start a transaction, then `SET LOCAL app.tenant_id = '<tenant-uuid>';` and **keep all SQL for that request inside that transaction**.  
> - If you use ORMs, inject `SET LOCAL` at txn start (e.g., a `before_query` hook).  
> - Avoid relying on session‑level state in transaction pooling.

### Schema‑per‑tenant option (brief)
- Create `tenant_<id>` schema per tenant; duplicate shared tables into each schema; use search path per request.  
- No RLS needed; stronger logical isolation but higher operational complexity.  
- Pooling impact: lower statement reuse, higher overhead with many schemas; avoid thousands+ without strong operational tooling.

---

## 3) Connection Pooling

**PgBouncer** in **transaction pooling** mode for high concurrency.

**Sizing rule of thumb**:  
- Postgres `max_connections` (e.g., 600)  
- Reserve for admin/monitoring (e.g., 50) → `~550` server connections available  
- App concurrency high? Set PgBouncer pool per DB/user (e.g., 100–300) but keep **total server connections ≤ 550**.  
- Use **prepared statements disabled** or **`server_reset_query = DISCARD ALL`**.

**Example `pgbouncer.ini`**
```ini
[databases]
app_prod = host=TARGET_DB_HOST port=5432 dbname=app_prod pool_size=300

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
ignore_startup_parameters = extra_float_digits
server_reset_query = DISCARD ALL
max_client_conn = 5000
default_pool_size = 300
min_pool_size = 50
reserve_pool_size = 20
reserve_pool_timeout = 5
query_timeout = 600
tcp_keepalive = 1
log_connections = 1
log_disconnections = 1
stats_period = 60
```

**App pattern per request/transaction**
```sql
begin;
set local app.tenant_id = '00000000-0000-0000-0000-000000000101'; -- tenant uuid
-- your queries here
commit;
```

---

## 4) Migration Choreography (Zero‑Downtime)

We’ll do **per‑tenant waves**:

**Phases**  
0. **Prep** target DB (tables, partitions, RLS, indexes, PgBouncer).  
1. **Register tenants** & source DSNs (secrets in vault).  
2. **FDW backfill** (pull) from source tenant DB → target `tenant_id` tables.  
3. **Source CDC**: install outbox triggers on each source; migrator applies outbox changes to target continuously.  
4. **Canary cutover**: pick small tenants → drain CDC lag to 0 → switch app reads/writes to target → enable **reverse‑CDC** for rollback safety window.  
5. **Wave cutover**: batch tenants by size; repeat.  
6. **Finalize**: disable reverse‑CDC; archive sources; optional read‑only window for a short period; decommission.

### 4.1 Prepare target (run once)

```sql
-- in target app_prod
create extension if not exists postgres_fdw;

-- (tables/partitions/RLS from earlier section)
-- Also add helpful indexes (some examples)
create index concurrently if not exists idx_users_tenant_email on users(tenant_id, email);
create index concurrently if not exists idx_orders_tenant_status on orders(tenant_id, status);
```

**Tenant registry** (insert known tenants)
```sql
insert into tenants(tenant_id, slug, source_dsn, state)
values
  ('00000000-0000-0000-0000-000000000101','acme','postgres://app:***@src-acme:5432/app','new'),
  ('00000000-0000-0000-0000-000000000202','globex','postgres://app:***@src-globex:5432/app','new');
```

> Store secrets in a secret manager and **never** check them into code. Use `CREATE USER MAPPING` with bound credentials.

### 4.2 For each tenant: create FDW and import foreign schemas

```sql
-- variables (conceptual)
-- :TENANT_UUID, :TENANT_SLUG, :TENANT_KEY (e.g., 101), and source DSN

-- Create a dedicated foreign server for the tenant
do $$
declare dsn text;
begin
  select source_dsn into dsn from tenants where tenant_id = :'TENANT_UUID'::uuid;
  execute format($fmt$
    create server if not exists src_%s foreign data wrapper postgres_fdw options (dbname '%s', host '%s', port '%s');
  $fmt$, :'TENANT_SLUG',
    regexp_replace(dsn, '.*dbname=([^ ]+).*', '\1'),
    regexp_replace(dsn, '.*@([^:/]+)[:/].*', '\1'),
    coalesce(nullif(regexp_replace(dsn, '.*:([0-9]+)/.*', '\1'), ''), '5432'));
end$$;

-- Create user mapping for a technical role (recommended: 'migrator')
-- Substitute proper credentials or rely on .pgpass/managed secrets
create user mapping if not exists for migrator server src_acme
  options (user 'app', password 'REDACTED');

-- Foreign schema per tenant to avoid name collisions
create schema if not exists fdw_t_101;

-- Import only the app tables you need (limit is safer)
import foreign schema public limit to (accounts, users, orders)
  from server src_acme into fdw_t_101;

-- (Optional) lock source tables to stabilize DDL during import; or enforce a schema freeze window
```

*(Repeat per tenant; you can automate with a small script that iterates tenants.)*

### 4.3 One‑time backfill (idempotent, chunked, online)

**Important**: ensure `(tenant_id, id)` composite PK is enforced in target. For backfill, do **upserts** and **chunk** by ranges to avoid long locks.

```sql
-- Example: accounts
insert into accounts (tenant_id, id, name, created_at, updated_at)
select :'TENANT_UUID'::uuid, a.id, a.name, a.created_at, a.updated_at
from fdw_t_101.accounts a
on conflict (tenant_id, id) do update
set name = excluded.name,
    updated_at = greatest(accounts.updated_at, excluded.updated_at);

-- users (respect FKs)
insert into users (tenant_id, id, account_id, email, created_at, updated_at)
select :'TENANT_UUID'::uuid, u.id, u.account_id, u.email, u.created_at, u.updated_at
from fdw_t_101.users u
on conflict (tenant_id, id) do update
set email = excluded.email,
    updated_at = greatest(users.updated_at, excluded.updated_at);

-- orders
insert into orders (tenant_id, id, account_id, total_cents, status, created_at, updated_at)
select :'TENANT_UUID'::uuid, o.id, o.account_id, o.total_cents, o.status, o.created_at, o.updated_at
from fdw_t_101.orders o
on conflict (tenant_id, id) do update
set total_cents = excluded.total_cents,
    status      = excluded.status,
    updated_at  = greatest(orders.updated_at, excluded.updated_at);
```

**Chunking** (example using `id` ranges):
```sql
-- Repeat with where clauses:
-- where id between 1 and 100000
-- where id between 100001 and 200000
-- … etc.
```

Update tenant state:
```sql
update tenants set state = 'backfilling', updated_at = now() where tenant_id = :'TENANT_UUID'::uuid;
```

### 4.4 CDC (change data capture) from source (zero downtime)

Add **outbox** tables & **row‑level triggers** on **each source** tenant DB to record mutations. These are **append‑only** and cheap.

**On each source tenant DB** (run once):

```sql
-- Outbox to capture DML in order
create table if not exists outbox_dml (
  id bigserial primary key,
  table_name text not null,
  op char(1) not null check (op in ('I','U','D')),
  pk bigint not null,
  row_data jsonb,             -- after image for I/U; before image for D (or null)
  occurred_at timestamptz not null default now()
);

create index if not exists idx_outbox_dml_occurred on outbox_dml(occurred_at, id);

-- Generic trigger function (adjust per table if needed)
create or replace function f_outbox_capture() returns trigger language plpgsql as $$
declare payload jsonb;
begin
  if (tg_op = 'INSERT') then
    payload := to_jsonb(new);
    insert into outbox_dml(table_name, op, pk, row_data)
      values (tg_table_name, 'I', new.id, payload);
    return new;
  elsif (tg_op = 'UPDATE') then
    payload := to_jsonb(new);
    insert into outbox_dml(table_name, op, pk, row_data)
      values (tg_table_name, 'U', new.id, payload);
    return new;
  elsif (tg_op = 'DELETE') then
    payload := to_jsonb(old);
    insert into outbox_dml(table_name, op, pk, row_data)
      values (tg_table_name, 'D', old.id, payload);
    return old;
  end if;
  return null;
end$$;

-- Attach to app tables
drop trigger if exists trg_outbox_accounts on accounts;
create trigger trg_outbox_accounts after insert or update or delete on accounts
for each row execute function f_outbox_capture();

drop trigger if exists trg_outbox_users on users;
create trigger trg_outbox_users after insert or update or delete on users
for each row execute function f_outbox_capture();

drop trigger if exists trg_outbox_orders on orders;
create trigger trg_outbox_orders after insert or update or delete on orders
for each row execute function f_outbox_capture();
```

**In the target DB**, ensure FDW imports the `outbox_dml` from each tenant source:

```sql
-- On target: extend FDW schema to include outbox_dml
import foreign schema public limit to (outbox_dml)
  from server src_acme into fdw_t_101;
```

**Migrator process** (idempotent upsert/apply loop):  
Pseudo‑SQL per tenant (run continuously; can be a worker):

```sql
-- pull in batches (e.g., 5_000 rows)
with batch as (
  select id, table_name, op, pk, row_data
  from fdw_t_101.outbox_dml
  where occurred_at <= now() - interval '2 seconds'
  order by id
  limit 5000
)
select apply_cdc(:TENANT_UUID, id, table_name, op, pk, row_data) from batch;
```

Define the `apply_cdc` function in target (one example; you can implement in application code as well):

```sql
create or replace function apply_cdc(tid uuid, ob_id bigint, tbl text, op char(1), pk bigint, row_data jsonb)
returns void language plpgsql as $$
begin
  if tbl = 'accounts' then
    if op = 'I' then
      insert into accounts (tenant_id,id,name,created_at,updated_at)
      values (tid, pk, row_data->>'name', (row_data->>'created_at')::timestamptz, (row_data->>'updated_at')::timestamptz)
      on conflict (tenant_id,id) do update
      set name = excluded.name,
          updated_at = greatest(accounts.updated_at, excluded.updated_at);
    elsif op = 'U' then
      update accounts set
        name = row_data->>'name',
        updated_at = greatest(updated_at, (row_data->>'updated_at')::timestamptz)
      where tenant_id = tid and id = pk;
    elsif op = 'D' then
      delete from accounts where tenant_id = tid and id = pk;
    end if;
  elsif tbl = 'users' then
    -- similar pattern; use upsert for I/U
    insert into users (tenant_id,id,account_id,email,created_at,updated_at)
    values (tid, pk, (row_data->>'account_id')::bigint, row_data->>'email',
            (row_data->>'created_at')::timestamptz, (row_data->>'updated_at')::timestamptz)
    on conflict (tenant_id,id) do update set
      email = excluded.email,
      updated_at = greatest(users.updated_at, excluded.updated_at);
    if op = 'D' then
      delete from users where tenant_id = tid and id = pk;
    end if;
  elsif tbl = 'orders' then
    -- same idea; idempotent
    insert into orders (tenant_id,id,account_id,total_cents,status,created_at,updated_at)
    values (tid, pk, (row_data->>'account_id')::bigint, (row_data->>'total_cents')::bigint,
            row_data->>'status', (row_data->>'created_at')::timestamptz, (row_data->>'updated_at')::timestamptz)
    on conflict (tenant_id,id) do update set
      total_cents = excluded.total_cents,
      status      = excluded.status,
      updated_at  = greatest(orders.updated_at, excluded.updated_at);
    if op = 'D' then
      delete from orders where tenant_id = tid and id = pk;
    end if;
  else
    -- extend for other tables
    null;
  end if;
end$$;
```

Set tenant state:
```sql
update tenants set state = 'cdc', updated_at = now()
where tenant_id = :'TENANT_UUID'::uuid;
```

### 4.5 Canary cutover (per‑tenant)

**Goal**: tenant experiences **no downtime** and **reads its own writes** after switch.

**Steps for a canary tenant**:

1. **Freeze DDL** for that tenant on source (no schema changes during cutover window).
2. **Drain CDC**: wait until `outbox lag == 0` for that tenant:
   ```sql
   -- compare target and source counts/checksums (see validation section)
   -- also ensure no unprocessed outbox in FDW:
   select count(*) from fdw_t_101.outbox_dml where occurred_at > now() - interval '2 seconds';
   ```
3. **Switch app** for this tenant:
   - **Reads/Writes** → target DB via PgBouncer.
   - Ensure app sets `SET LOCAL app.tenant_id = '<tenant-uuid>'` per txn.
4. **Enable reverse‑CDC** (temporary, safety period):
   - Add outbox/triggers on **target** for tenant‑scoped tables (same as source’s pattern, but record to `outbox_dml_target`).
   - Configure a **reverse migrator** to apply **target→source** (only for that cutover tenant), so if rollback happens, **no data is lost**.
5. **Observe** 30–120 minutes: latency, error rates, isolation tests (RLS leakage tests), and business KPIs.
6. **If healthy**: mark tenant `postcutover`.

Commands to “flip” the app (pattern; actual wiring depends on your config/feature flags):

```bash
# Example using a feature flag service or config map keyed by tenant slug
# set TENANT_ROUTES[acme] = target
```

### 4.6 Wave cutover (repeat for all tenants)
- Batch tenants by size.  
- Parallelize CDC workers, but keep an eye on target write IOPS and vacuum.  
- After each wave’s soak period, disable reverse‑CDC for the wave.

### 4.7 Finalize
- Mark all tenants `done`.  
- Disable/Drop source outbox triggers.  
- Keep the source DBs **read‑only** for a week (optional) before decommission.  
- Backup & archive snapshots.

---

## 5) Step‑by‑Step Runbook (precise commands)

> Replace placeholders (`TARGET_DB_HOST`, `TENANT_UUID`, `TENANT_SLUG`, etc.) and put secrets in a proper secret manager.

### 5.1 Target initialization

```sql
-- Connect to target: psql "host=TARGET_DB_HOST dbname=app_prod user=admin sslmode=require"

create extension if not exists postgres_fdw;

-- Create app tables, partitions, and RLS (from Section 2)

-- Performance tweaks (example; verify for your infra)
alter system set max_wal_size = '8GB';
alter system set maintenance_work_mem = '2GB';
select pg_reload_conf();
```

### 5.2 Register tenants

```sql
insert into tenants(tenant_id, slug, source_dsn, state)
values ('00000000-0000-0000-0000-000000000101','acme','postgres://app:***@src-acme:5432/app','new');
```

### 5.3 Create FDW per tenant and import tables

```sql
-- As admin on target
create server if not exists src_acme foreign data wrapper postgres_fdw options (host 'src-acme', dbname 'app', port '5432');
create user mapping if not exists for migrator server src_acme options (user 'app', password 'REDACTED');
create schema if not exists fdw_t_101;
import foreign schema public limit to (accounts, users, orders, outbox_dml)
  from server src_acme into fdw_t_101;
```

### 5.4 Backfill (idempotent, chunked)

```sql
-- Repeat for each table; illustrate users table with chunk
begin;
-- chunk 1..100000
insert into users (tenant_id,id,account_id,email,created_at,updated_at)
select '00000000-0000-0000-0000-000000000101'::uuid, id, account_id, email, created_at, updated_at
from fdw_t_101.users where id between 1 and 100000
on conflict (tenant_id,id) do update
set email = excluded.email,
    updated_at = greatest(users.updated_at, excluded.updated_at);
commit;
```

### 5.5 Install source CDC triggers (on each **source** DB)

```sql
-- Connect to source tenant DB (e.g., src-acme:5432/app)
-- Create outbox and triggers (from Section 4.4)
```

### 5.6 Start migrator (source→target)

Implement as a service or cron job; pseudo‑SQL batch loop:

```sql
-- On target, per tenant in state='cdc'
with batch as (
  select id, table_name, op, pk, row_data
  from fdw_t_101.outbox_dml
  where occurred_at <= now() - interval '2 seconds'
  order by id
  limit 5000
)
select apply_cdc('00000000-0000-0000-0000-000000000101'::uuid, id, table_name, op, pk, row_data)
from batch;
```

**Mark state**:
```sql
update tenants set state='cdc', updated_at=now() where tenant_id='00000000-0000-0000-0000-000000000101';
```

### 5.7 Canary cutover

**Drain & verify:**
```sql
-- ensure no pending outbox
select count(*) from fdw_t_101.outbox_dml where occurred_at > now() - interval '2 seconds';
-- run validation queries (see Section 6)
```

**Flip app** routing for that tenant to target.

**Enable reverse‑CDC (target→source)** for safety period:

_On target:_
```sql
create table if not exists outbox_dml_target (
  id bigserial primary key,
  table_name text not null,
  op char(1) not null,
  pk bigint not null,
  row_data jsonb,
  tenant_id uuid not null,
  occurred_at timestamptz not null default now()
);

create or replace function f_outbox_capture_target() returns trigger language plpgsql as $$
begin
  if (tg_op = 'INSERT') then
    insert into outbox_dml_target(table_name, op, pk, row_data, tenant_id)
    values (tg_table_name, 'I', new.id, to_jsonb(new), new.tenant_id);
    return new;
  elsif (tg_op = 'UPDATE') then
    insert into outbox_dml_target(table_name, op, pk, row_data, tenant_id)
    values (tg_table_name, 'U', new.id, to_jsonb(new), new.tenant_id);
    return new;
  elsif (tg_op = 'DELETE') then
    insert into outbox_dml_target(table_name, op, pk, row_data, tenant_id)
    values (tg_table_name, 'D', old.id, to_jsonb(old), old.tenant_id);
    return old;
  end if;
  return null;
end$$;

-- Attach triggers (conditionally for the canary tenant via WHEN clause)
drop trigger if exists trg_outbox_tgt_accounts on accounts;
create trigger trg_outbox_tgt_accounts after insert or update or delete on accounts
for each row when (new.tenant_id = '00000000-0000-0000-0000-000000000101'::uuid or old.tenant_id = '00000000-0000-0000-0000-000000000101'::uuid)
execute function f_outbox_capture_target();

-- Repeat for users, orders
```

_Reverse apply from target→source_ (process rows in `outbox_dml_target` for the single tenant and execute DML against the source via app/migrator). After soak period (e.g., 1–2 hours), **disable** these target triggers for that tenant.

Mark tenant:
```sql
update tenants set state='postcutover', updated_at=now()
where tenant_id='00000000-0000-0000-0000-000000000101';
```

### 5.8 Wave cutover
- Repeat 5.7 for tenant batches; parallelize cautiously.

### 5.9 Finalize
- Disable source and target outbox triggers; drop reverse‑CDC.  
- Mark tenants `done`.  
- Keep sources read‑only for a week (optional), then decommission.

---

## 6) Data Validation Tactics

Run **before** and **after** cutover, and periodically during CDC, **per tenant** and **per table**.

### 6.1 Deterministic counts (per tenant)

```sql
-- Source count (via FDW)
select count(*) from fdw_t_101.users;

-- Target count
select count(*) from users where tenant_id='00000000-0000-0000-0000-000000000101';
```

### 6.2 Checksums (fast, approximate but stable)
Use hash on stable columns (pk + updated_at + critical cols). Postgres built‑ins like `hashtext`, `hashint8` are available.

```sql
-- Source
select
  count(*)                        as n,
  sum(hashint8(id))               as h_id,
  sum(hashtext(coalesce(email,''))) as h_email,
  sum((extract(epoch from updated_at))::bigint) as h_ts
from fdw_t_101.users;

-- Target
select
  count(*)                        as n,
  sum(hashint8(id))               as h_id,
  sum(hashtext(coalesce(email,''))) as h_email,
  sum((extract(epoch from updated_at))::bigint) as h_ts
from users where tenant_id='00000000-0000-0000-0000-000000000101';
```

For critical tables, add **stratified** checks (per day/per account_id).

### 6.3 Referential integrity (RI)

```sql
-- Users → Accounts
select count(*) as broken_fk
from users u
left join accounts a
  on a.tenant_id=u.tenant_id and a.id=u.account_id
where u.tenant_id='...tenant...' and a.id is null;
```

### 6.4 Spot row‑by‑row sampling
```sql
-- Pick random 100 ids from source, compare fields
with sample as (
  select id from fdw_t_101.users tablesample system (1) limit 100
)
select s.id, src.email as src_email, tgt.email as tgt_email
from sample s
join fdw_t_101.users src on src.id = s.id
left join users tgt on tgt.tenant_id='...tenant...' and tgt.id=s.id
where src.email is distinct from tgt.email;
```

### 6.5 RLS isolation tests
- Run a suite that attempts cross‑tenant reads with mismatched `SET LOCAL app.tenant_id` and ensure **0 rows**.  
- Verify no table is missing `ENABLE ROW LEVEL SECURITY` and policy.

```sql
-- Audit RLS
select relname, relrowsecurity, relforcerowsecurity
from pg_class c join pg_namespace n on n.oid=c.relnamespace
where nspname='public' and relkind='r';

-- Attempt cross-tenant read (should return 0)
begin;
set local app.tenant_id='TENANT_A_UUID';
select count(*) from orders where tenant_id='TENANT_B_UUID'; -- expect 0 visible rows even if data exists
rollback;
```

### 6.6 Business metrics parity
- Revenue totals, active users, last 7d order counts—**compare source vs target** for the tenant before cutover.

---

## 7) Failure Playbook (What could go wrong & how to fix fast)

### A) CDC lag grows (migrator can’t keep up)
**Symptoms**: increasing `fdw_t_*/outbox_dml` backlog; user sees stale data in target pre‑cutover.  
**Actions**:
- Scale migrator workers; shrink batch size to reduce lock pressure; increase `work_mem` for apply.  
- Create missing indexes on target (e.g., `orders(tenant_id,id)` is PK already; add hot filters like `status`).  
- Throttle source write bursts temporarily (rate limit).  
- Verify network latency; colocate target with sources during migration if possible.

### B) RLS misconfiguration (data leakage risk)
**Symptoms**: cross‑tenant reads return rows.  
**Actions**:
- Immediately **flip tenant reads back to source** using routing flag.  
- `revoke all on all tables in schema public from app_user;` (if needed)  
- Audit RLS policies (Section 6.5) and ensure app sets `SET LOCAL app.tenant_id`.  
- Add **test gates** in CI to reject tables without RLS.

### C) Connection pool exhaustion / errors under load
**Symptoms**: `too many connections`, timeouts, spikes.  
**Actions**:
- Increase PgBouncer `default_pool_size` cautiously; lower client concurrency.  
- Ensure app uses **short transactions**; no session state.  
- Verify Postgres `max_connections`, `shared_buffers`, and CPU/IO headroom.  
- Examine slow queries → add indexes, analyze, `vacuum (analyze)`.

### D) Sequence or PK collisions
**Symptoms**: inserts fail on target due to duplicate `(tenant_id,id)` when app starts writing new records.  
**Actions**:
- Ensure **composite PK** is `(tenant_id,id)` and app sets correct tenant_id.  
- If global unique ids desired, switch to UUIDv7 or K‑sorted ids; update app before cutover.

### E) FDW errors / DDL drift on source
**Symptoms**: backfill fails due to schema change on source.  
**Actions**:
- Enforce **DDL freeze** during migration windows; if unavoidable, apply the same DDL to target and adjust migrator mapping.  
- Re‑`import foreign schema` after schema updates.

### F) Outbox triggers overhead
**Symptoms**: latency increases on source writes.  
**Actions**:
- Ensure outbox rows are small; keep `row_data` minimal.  
- Index outbox by `(occurred_at, id)`; vacuum frequently on source.  
- If very high write rate, switch the busiest tenants first to shorten the CDC window.

### G) Target bloat / vacuum
**Symptoms**: table bloat, slow queries.  
**Actions**:
- Enable aggressive autovacuum for large partitions:
  ```sql
  alter table orders_p0 set (autovacuum_vacuum_scale_factor=0.05, autovacuum_analyze_scale_factor=0.02);
  ```
- Reindex concurrently hot indexes off‑hours.

### H) Cutover goes bad for a tenant
**Symptoms**: errors or unacceptable performance post‑flip.  
**Immediate rollback**:
- Route the tenant back to **source** for reads/writes (toggle route flag).  
- Because **reverse‑CDC** is enabled during the safety window, data written to target is mirrored back to source – no loss.  
- Keep CDC running source→target to keep states in sync while you remediate.  
- Once fixed, retry cutover.

### I) Logical corruption / business invariants violated
**Symptoms**: validation diffs in critical metrics.  
**Actions**:
- Pause cutover; isolate the tenant; run **differential compare** and re‑sync the affected rows from source using FDW.  
- Add **idempotent repair scripts**.

---

## 8) Cutover Plan (Timeline & Commands)

### T‑14 to T‑7 days
- Provision target DB, PgBouncer, monitoring.  
- Create tables/partitions and RLS.  
- Register tenants in `tenants`.  
- Implement app support for `SET LOCAL app.tenant_id` per txn.

### T‑7 to T‑2 days
- For each tenant: Create FDW, backfill (idempotent).  
- Install outbox triggers (source).  
- Start CDC migrator (source→target).  
- Continuous validation (counts/checksums).

### T‑1 day
- Canary tenant: drain CDC, validate, flip reads/writes to target.  
- Enable reverse‑CDC (target→source) for canary. Soak 1–2 hours, then disable reverse‑CDC if healthy.

### T day – T+7
- Wave cutovers (repeat process).  
- Keep reverse‑CDC enabled for ~1–2 hours per tenant post‑flip; then disable.  
- Monitor RLS leakage tests, performance, KPIs.

### T+7
- All tenants in `postcutover` → set to `done`.  
- Disable and remove CDC triggers on both sides.  
- Sources read‑only → archive → decommission.

**Minimal command snippets to flip a tenant**

- **App routing flag** (conceptual; implement in your config/feature flags):
  ```bash
  # pseudo
  feature_flag set tenant_route.acme target
  ```
- **Disable reverse CDC for tenant after soak**
  ```sql
  -- On target: drop or disable the WHEN-tenant triggers
  drop trigger if exists trg_outbox_tgt_accounts on accounts;
  drop trigger if exists trg_outbox_tgt_users on users;
  drop trigger if exists trg_outbox_tgt_orders on orders;
  ```

---

## 9) Rollback Plan (Per‑tenant and Global)

**During tenant cutover safety window**:
- **Immediate rollback**: set route back to **source**; reverse‑CDC ensured source is current.  
- Keep source→target CDC on to avoid divergence.  
- After remediation, try again.

**After safety window (reverse‑CDC off)**:
- If rollback is needed later, temporarily re‑enable reverse‑CDC **from target to source**, backfill any gap via FDW (target→source), drain to zero, then flip route back.

**Global rollback**:
- If systemic issue, roll back tenants still in source immediately by halting further cutovers.  
- For tenants already flipped, follow per‑tenant rollback steps above.

**Data protection**:
- Retain frequent snapshots (e.g., hourly PITR, daily base backups) on target during migration waves.  
- Use WAL archiving and verify restore rehearsals.

---

## 10) Final Self‑Check (Readiness)

**People & Process**
- [ ] On‑call rotation established for migration windows.  
- [ ] Runbooks printed/accessible; DR contact tree set.

**App**
- [ ] Every request/transaction sets `SET LOCAL app.tenant_id` **before** running queries.  
- [ ] No reliance on session state due to transaction pooling.  
- [ ] Feature flag for **per‑tenant routing** (source vs target) is live and tested.

**Database (Target)**
- [ ] RLS enabled on all tenant tables; policies verified.  
- [ ] Composite PK `(tenant_id, id)` in place for all tables.  
- [ ] Hash partitions created (e.g., 32) and attached.  
- [ ] Critical indexes present and `ANALYZE` completed.  
- [ ] Autovacuum settings tuned for large partitions.  
- [ ] Backups and PITR verified (test restore).

**Database (Source)**
- [ ] Outbox triggers installed and tested (I/U/D).  
- [ ] Outbox indexes exist; autovacuum healthy.  
- [ ] DDL freeze policy agreed for migration window.

**Connectivity & Pooling**
- [ ] PgBouncer configured in transaction mode; load tested.  
- [ ] `server_reset_query = DISCARD ALL` verified.  
- [ ] Connection budgets (client vs server) verified under peak load.

**Migration tooling**
- [ ] FDW servers & schemas created per tenant; permissions correct.  
- [ ] Backfill jobs idempotent and chunked; no long‑running locks.  
- [ ] CDC migrator processes (src→tgt) monitored; metrics/alerts on **lag**.  
- [ ] Reverse‑CDC (tgt→src) capability tested on staging.

**Validation**
- [ ] Count & checksum queries scripted; pass within tolerance.  
- [ ] RI checks pass; no orphan references.  
- [ ] RLS leakage tests pass (forced attempts return 0).  
- [ ] Business metrics parity (revenue/orders/users) within agreed thresholds.

**Cutover**
- [ ] Canary tenant passes soak; rollback drill executed successfully.  
- [ ] Wave plan sequenced (low → high traffic).  
- [ ] Communication plan to stakeholders in place.

**Post‑migration**
- [ ] Sources read‑only for grace period; archives retained.  
- [ ] Observability dashboards for target: TPS, latency, deadlocks, bloat, cache hit ratio.  
- [ ] Cost/perf regression review scheduled.

---

### Notes & Practical Tips

- Prefer **UUIDv7** (or ULID) for new globally unique keys if you plan to reduce reliance on per‑tenant numeric `id`.  
- Keep CDC windows **short**: the longer the dual writes/CDC, the higher the risk of drift.  
- If a subset of tenants requires **hard isolation**, use the **hybrid** approach: keep those in **schema‑per‑tenant** on the same cluster (or a separate DB) and route accordingly—you can still leverage PgBouncer and shared operational tooling.  
- Always test on a **staging environment** with production‑like data volumes: validate backfill rates, CDC throughput, and cutover scripts.

This plan provides **zero‑downtime**, **per‑tenant cutover**, robust **validation**, and a **safe rollback** path—all with precise operational steps and commands you can adapt directly to your environment.