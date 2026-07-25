# shopify-etl

Shopify → PostgreSQL ETL service for Saffron Road. Syncs customers, orders, products, variants, catalogs, line items, and refunds from the Shopify Admin API into an external PostgreSQL cluster on a configurable cron schedule.

**Image:** `gdelapierre/shopify-etl-nodb:<version>`
**HTTP port:** `3002` (manual trigger endpoint)
**Env file:** `docker/stack-env/shopify-etl/shopify-etl.env`

---

## Setup run order

Three SQL scripts handle the full lifecycle. Run them in order on a new cluster:

```
postgres superuser (one-time)
  └── utils/00_cluster_bootstrap.sql   → creates dbadmin role + cluster grants
        └── utils/01_setup.sql          → ETL database, loader/n8n roles, schema
                                          (automated via RUN_DB_INIT=true)
        └── utils/00-createDB.sql       → reporting database + saffronroad_readonly role
                                          (run manually in pgAdmin or psql)
```

### Step 1 — cluster bootstrap (once per cluster, as `postgres`)

```bash
psql -h <host> -U postgres -d postgres \
     -v dbadmin_password='<password>' \
     -f utils/00_cluster_bootstrap.sql
```

Creates `dbadmin` with `CREATEDB`, `CREATEROLE`, and the `pg_monitor` / `pg_read_all_data` / `pg_write_all_data` / `pg_signal_backend` grants. Skip if `dbadmin` already exists on the cluster.

### Step 2 — ETL database (automated)

Set `RUN_DB_INIT=true` in the env and start the container — `01_setup.sql` runs automatically as `dbadmin`. See [Database initialisation](#database-initialisation-run_db_init) below.

### Step 3 — reporting database (manual)

Run `utils/00-createDB.sql` as `dbadmin` in pgAdmin or psql. Creates the `app-saffronroad-shopify` database and `saffronroad_readonly` role.

---

## Other prerequisites

- CA certificate bundle at `/data/compose/shopify-etl/certs/dlplab-root-ca.pem` on the Docker host (intermediate + root chain from Vault PKI)

---

## Database initialisation (`RUN_DB_INIT`)

| `RUN_DB_INIT` | Behaviour |
|---|---|
| `false` (default) | Container starts the ETL service directly — assumes schema already exists |
| `true` | Runs `utils/init-db.sh` → `utils/01_setup.sql` before starting the ETL service |

The init script:
1. Waits for PostgreSQL to be reachable
2. Creates `loader` and `n8n` roles — **only if** the corresponding password env vars are set; warns and skips if they are empty (safe when roles already exist)
3. Creates the database if missing
4. Creates all tables, indexes, and applies grants

Set `RUN_DB_INIT=true` on first deploy to a new database. Set back to `false` afterwards to skip the overhead on subsequent restarts.

---

## Environment variables

### Shopify

Two auth modes (from image 1.8.0): **client credentials grant** (preferred — Dev Dashboard app; token auto-refreshed every 24h) or a **static admin token** (legacy fallback). Client credentials win when both are set.

| Variable | Default | Description |
|---|---|---|
| `SHOPIFY_STORE_DOMAIN` | required | e.g. `yourstore.myshopify.com` |
| `SHOPIFY_API_VERSION` | `2025-10` | Shopify Admin API version |
| `SHOPIFY_CLIENT_ID` | — | Dev Dashboard app client ID (client credentials grant) |
| `SHOPIFY_CLIENT_SECRET` | — | Dev Dashboard app secret, `shpss_…` (client credentials grant) |
| `SHOPIFY_ADMIN_ACCESS_TOKEN` | — | Static custom-app admin token — fallback when client credentials are unset |

### PostgreSQL (ETL runtime user)

| Variable | Default | Description |
|---|---|---|
| `PGHOST` | required | Cluster hostname |
| `PGPORT` | `5432` | |
| `PGDATABASE` | required | Target database name |
| `PGUSER` | required | Runtime user (e.g. `loader`) |
| `PGPASSWORD` | required | |
| `PGSSLMODE` | `prefer` | Set to `require` for production |
| `PGSSL_REJECT_UNAUTHORIZED` | `true` | Reject untrusted server certs |
| `PGSSLROOTCERT` | — | Path to CA bundle inside container (set to `/etc/ssl/dlplab-root-ca.pem`) |

### Database initialisation (only needed when `RUN_DB_INIT=true`)

| Variable | Default | Description |
|---|---|---|
| `RUN_DB_INIT` | `false` | Set to `true` on first deploy |
| `DB_ADMIN_USER` | `dbadmin` | Admin role used to run the init script |
| `DB_ADMIN_PASSWORD` | required | Password for `DB_ADMIN_USER` |
| `DB_ADMIN_DB` | `postgres` | Admin database to connect to during init |
| `DB_LOADER_PASSWORD` | — | Password for `loader` role creation (skip if role exists) |
| `DB_N8N_PASSWORD` | — | Password for `n8n` role creation (skip if role exists) |

### Scheduler

| Variable | Default | Description |
|---|---|---|
| `ETL_SCHEDULE` | `0 * * * *` | Cron expression (hourly) |
| `RUN_ON_STARTUP` | `false` | Run a full sync immediately on container start |
| `TZ` | `Australia/Brisbane` | Timezone for cron scheduling |

---

## CA certificate

The container expects the DLPLab CA bundle (intermediate + root) at `/etc/ssl/dlplab-root-ca.pem`, injected via volume mount:

```
/data/compose/shopify-etl/certs/dlplab-root-ca.pem:/etc/ssl/dlplab-root-ca.pem:ro
```

Build the bundle from Vault PKI:

```bash
curl -s https://vault.prod.dlplab.app/v1/pki_int/ca/pem > /data/compose/shopify-etl/certs/dlplab-root-ca.pem
curl -s https://vault.prod.dlplab.app/v1/pki_root/ca/pem >> /data/compose/shopify-etl/certs/dlplab-root-ca.pem
```

> The PostgreSQL server certificate is short-lived (3-day TTL). Verify Vault auto-renewal is active to avoid connection failures.

---

## Manual trigger

```bash
curl -X POST http://<host>:3002/sync
```
