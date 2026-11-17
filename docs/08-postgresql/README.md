# 08 — PostgreSQL

> A reliable database that stores your apps' data in tables.

## 1. What is it?

**PostgreSQL** ("Postgres") is a free, powerful **relational database**. Data is stored in **tables** (rows and columns), and you talk to it with **SQL**.

Comparison: a very organized **filing cabinet** with labeled drawers (databases), folders (tables), and papers (rows). Only people with the right key (user/password) can open certain drawers.

## 2. Why this homelab uses it

Your K3s apps (and maybe n8n) need somewhere safe to store data. Postgres runs in **LXC 105** (`192.168.0.225`), outside Kubernetes, so the data is simple to back up and doesn't move around.

## 3. Key words

| Word | Meaning |
|---|---|
| **Database** | A separate collection of tables. One per app |
| **Role / user** | A login. One per app, with access only to its own DB |
| **SQL** | The language: `SELECT * FROM users;` |
| **`psql`** | Command-line client |
| **`postgresql.conf`** | Main settings (memory, which IPs to listen on) |
| **`pg_hba.conf`** | **Who** may connect **from where** with **which** auth method |
| **`pg_dump`** | Export a database to a file (logical backup) |
| **`pg_restore` / `psql -f`** | Import a backup |
| **Connection string** | `postgresql://app1:PASSWORD@192.168.0.225:5432/app1` |
| **scram-sha-256** | Secure password method — use this, not `md5` or `trust` |

## 4. How it works here

```text
K3s pod (on worker .222/.223) ──TCP 5432──► LXC 105 postgres
   (pod IP is translated to the worker IP, so pg_hba allows .222 and .223)
```

## 5. Step by step for this project

1. **Install** (inside LXC 105: `pct enter 105` or `ssh postgres`):
   ```bash
   apt update && apt install -y postgresql zstd
   psql --version          # Ubuntu 24.04 ships PostgreSQL 16
   ```
2. **Listen on the LAN** — edit `/etc/postgresql/16/main/postgresql.conf`:
   ```text
   listen_addresses = 'localhost,192.168.0.225'
   shared_buffers = 512MB          # ~1/4 of RAM given to Postgres
   effective_cache_size = 2GB
   ```
3. **Allow only the K3s workers** — add to `/etc/postgresql/16/main/pg_hba.conf`:
   ```text
   # TYPE  DATABASE  USER   ADDRESS            METHOD
   host    app1      app1   192.168.0.222/32   scram-sha-256
   host    app1      app1   192.168.0.223/32   scram-sha-256
   ```
4. **Create a user and database for one app:**
   ```bash
   sudo -u postgres psql
   ```
   ```sql
   CREATE ROLE app1 LOGIN PASSWORD 'use-a-long-random-password';
   CREATE DATABASE app1 OWNER app1;
   \q
   ```
5. **Restart and test from a worker:**
   ```bash
   systemctl restart postgresql
   # on worker1:
   psql "postgresql://app1@192.168.0.225:5432/app1" -c 'select 1;'
   ```
6. **Nightly backup** — `/etc/cron.d/pgdump`:
   ```text
   0 2 * * * postgres pg_dumpall | zstd > /var/backups/pg/all-$(date +\%F).sql.zst && find /var/backups/pg -mtime +3 -delete
   ```
   (Create `/var/backups/pg` owned by `postgres` first.) restic uploads this folder to R2 ([25](../25-backups/README.md)).
7. **Practice a restore** into a test database at least once.
8. **Monitoring:** install `postgres_exporter` so Prometheus can read DB health ([21](../21-prometheus/README.md)).

## 6. Useful commands

```bash
sudo -u postgres psql                 # open SQL shell as admin
\l      # list databases
\du     # list users
\c app1 # connect to database app1
\dt     # list tables
\q      # quit

systemctl status postgresql
journalctl -u postgresql -f
tail -f /var/log/postgresql/postgresql-16-main.log

pg_dump -U app1 -h localhost app1 > app1.sql        # backup one DB
psql -U app1 -h localhost app1 < app1.sql           # restore
```

## 7. Common mistakes

- `listen_addresses = '*'` + `host all all 0.0.0.0/0 trust` → **anyone can log in**. Never.
- One shared `postgres` superuser for all apps. Use one role per app.
- Backups that were **never restored** — you don't know if they work.
- Publishing port 5432 to the internet. Keep it private.
- Running out of disk — watch DB size in Grafana.

## 8. Check yourself

1. Which file decides which IPs may connect?
2. Why does `pg_hba.conf` allow the worker IPs and not pod IPs?
3. What command makes a backup of one database?

<details><summary>Answers</summary>

1. `pg_hba.conf`
2. Traffic leaving the cluster to the LAN is translated (masqueraded) to the worker node's IP.
3. `pg_dump`

</details>

## 9. Learn more

- PostgreSQL docs: <https://www.postgresql.org/docs/current/>
- `pg_hba.conf`: <https://www.postgresql.org/docs/current/auth-pg-hba-conf.html>
- SQL tutorial: <https://www.postgresqltutorial.com/>
