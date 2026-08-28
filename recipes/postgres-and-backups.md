---
num: "09"
title: "PostgreSQL and Backups"
desc: "Running PostgreSQL for a homelab, with hourly off-site backups, tiered retention, and a restore you have actually tested."
tags: ["PostgreSQL", "Backups", "Cloudflare R2", "Disaster Recovery"]
color: accent
pubDate: 2026-05-17
---

## Context

Your apps are stateless. Something still has to hold the state, and that something is a database. That's twelve-factor working exactly as designed — stateless processes, persistent data in a backing service.

The homelab's twist is that you *operate* that backing service yourself instead of renting a managed one. Twelve-factor is silent on that choice, and it's a fine choice — but it hands you a bill: the durability a managed cloud database would have quietly provided is now entirely your job. And the risk it covers is simple and total: **lose the data, lose everything.** So this recipe is really two recipes. Running PostgreSQL is the easy half. Backing it up — off the machine, automatically, with a restore you've actually tested — is the half that matters.

## Run PostgreSQL as a container

PostgreSQL runs as a plain Docker container on the host, *outside* the Kubernetes cluster. That's deliberate: the cluster is disposable and gets rebuilt; the database should not share that fate. Keeping it as a host container means recreating the cluster never touches the data.

```bash
docker run -d \
  --name homelab-postgres \
  -e POSTGRES_USER=appuser \
  -e POSTGRES_PASSWORD="$PG_PASSWORD" \
  -e POSTGRES_DB=appdb \
  -v "$HOME/homelab/postgres:/var/lib/postgresql/data" \
  -p 5432:5432 \
  --restart unless-stopped \
  postgres:18-alpine
```

The details that matter:

- **`-v "$HOME/homelab/postgres:..."`** — the data lives in a host directory. It **must** be under your home directory. The [cluster-host recipe](/homelab/cluster-host) covers why: Colima only shares certain host paths into its Linux VM, and a path like `/data/...` may simply not exist in the VM, leaving you with an empty directory or a container that won't start. Home directory works. This is the single most common way to lose a homelab database before it's even done anything — to a *path bug*. Get it right here.
- **`--restart unless-stopped`** — the container comes back after a host reboot on its own.
- **`-p 5432:5432`** — the port is published on the host so the cluster can reach it.
- Fetch `$PG_PASSWORD` from your vault (the [secrets recipe](/homelab/1password-secrets)), don't type it — see that recipe's warning about secrets on the command line.

Wrap this in a script that pulls the password from the vault, checks whether the container already exists, and waits for `pg_isready` before declaring success. Then starting the database is one idempotent command.

## Connecting from the cluster

Pods reach the database at **`host.k3d.internal:5432`** — a hostname k3d wires into the cluster that resolves to the Docker host. From inside any pod, the connection string is:

```
Host=host.k3d.internal;Port=5432;Database=appdb;Username=appuser;Password=...
```

That connection string is a secret — it's synced from the vault like any other (the [secrets recipe](/homelab/1password-secrets)).

> `host.k3d.internal` is also the homelab's most common failure source. If it goes stale — after a reboot or a sleep/wake — pods get database timeouts that look like a database outage but aren't. Same if `k3d cluster start` hangs on the CoreDNS hostAliases line and only the DB-backed sites stay down. The [cluster-host recipe](/homelab/cluster-host)'s troubleshooting has the fix; if database connections start timing out cluster-wide and the database itself is fine, look there first.

## Backups: the part that matters

A backup on the same machine as the database is not a backup — it's a second copy of a single point of failure. When the Mac Mini's disk dies, they die together. Backups have to leave the building.

The setup: an hourly Kubernetes `CronJob` that runs `pg_dump` and uploads the result to **Cloudflare R2** (S3-compatible object storage, no egress fees, generous free tier). R2 credentials come from the vault.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
  namespace: apps
spec:
  schedule: "0 * * * *"          # top of every hour
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: postgres:18-alpine   # pg_dump must match the server version
              command: ["/bin/sh", "/scripts/backup.sh"]
              envFrom:
                - secretRef:
                    name: db-backup-secrets
          # ...mount the backup script, etc.
```

The backup script: `pg_dump -Fc` against `host.k3d.internal`, then upload to R2 with the AWS CLI pointed at R2's endpoint.

### Tiered retention

Hourly backups forever would be thousands of files and a slowly growing bill. Instead, age backups through tiers:

| Tier | Kept | Covers |
|------|------|--------|
| Hourly | last 24 | the last day, hour by hour |
| Daily | last 30 | the last month |
| Monthly | last 3 | the last quarter |

Total: roughly 90 days of history in well under a hundred files. The promotion logic, run as part of the backup job: at midnight, copy the day's backup into the daily tier; on the first of the month, copy one into the monthly tier; trim each tier to its limit.

> **First-run gotcha:** promotion *copies between* tiers, so it quietly assumes the tier folders already exist. On a brand-new bucket they don't, and the first promotions can no-op or error. Seed the structure once by hand — copy any existing backup into each tier path so `hourly/`, `daily/`, and `monthly/` all exist — and the automation is happy from then on.

### Verify it

A backup job that *runs* is not a backup job that *works*. Check both:

```bash
kubectl -n apps get cronjob db-backup
kubectl -n apps create job --from=cronjob/db-backup db-backup-test
kubectl -n apps logs -f -l job-name=db-backup-test
```

Then confirm the file actually reached R2:

```bash
aws s3 ls s3://homelab-backups/hourly/ --endpoint-url "$R2_ENDPOINT"
```

## Disaster recovery: the restore

The backup half is worthless without this half. **A backup you have never restored is a hope, not a backup.** Do a restore drill — into a scratch database — at least once, so the procedure is something you've *done*, not something you'll be reading for the first time during an actual incident.

The procedure:

**1. Stop writes.** Scale the apps to zero so nothing writes mid-restore:

```bash
kubectl -n apps scale deploy --all --replicas=0
```

**2. Download a backup:**

```bash
aws s3 cp s3://homelab-backups/hourly/<backup-file> ./ --endpoint-url "$R2_ENDPOINT"
```

**3. Restore.** Drop and recreate the database, then `pg_restore`:

```bash
docker exec homelab-postgres psql -U appuser -c "DROP DATABASE IF EXISTS appdb;"
docker exec homelab-postgres psql -U appuser -c "CREATE DATABASE appdb;"
docker exec -i homelab-postgres pg_restore -U appuser -d appdb \
  --no-owner --no-privileges --verbose < <backup-file>
```

> Restoring into a *fresh scratch database* first (`appdb_restore_test`) instead of dropping the live one is the safe way to run a drill — and the safe way to verify a backup before you trust it in a real incident.

**4. Verify**, then **scale the apps back up:**

```bash
docker exec -it homelab-postgres psql -U appuser -d appdb \
  -c "SELECT relname, n_live_tup FROM pg_stat_user_tables ORDER BY n_live_tup DESC LIMIT 10;"
kubectl -n apps scale deploy --all --replicas=1
```

Worst-case data loss with hourly backups is one hour. That's a deliberate, stated tradeoff — fine for a homelab. If you ever need tighter, that's continuous archiving (WAL shipping), which is a real step up in complexity; cross that bridge when an hour of loss actually hurts.

## Admin processes: migrations as Jobs

Schema migrations are the textbook twelve-factor "admin process" — a one-off task, run against the same image and config as the app, with a clear beginning and end. Run them as a Kubernetes `Job`, never as logic bolted into app startup:

```bash
kubectl -n apps apply -f migration-job.yaml
kubectl -n apps logs -f job/db-migrate
```

A `Job` is its own object with its own status and its own logs. When a migration fails, you have exactly one thing to point at — not a crash-looping app that's failing for a reason you have to *infer*. Take a backup before any migration that isn't trivially reversible; the restore procedure above is your undo button.

## When it breaks

### Symptom: `pg_dump`/`pg_restore` fails with a version error

The classic. `pg_dump` refuses to dump from a server *newer* than itself. If your backup job's image, or your laptop's client tools, are an older PostgreSQL major version than the running server, every dump fails.

The rule: **the client major version must be ≥ the server major version.** Keep them equal. Two places this bites:

- **The backup CronJob** — its image must match the server. If the database is `postgres:18`, the job image is `postgres:18-alpine`, not whatever was current last year.
- **Your laptop** — `brew install postgresql@18`, and don't forget the keg-only PATH step (the [cluster-host recipe](/homelab/cluster-host) covers it) or you'll silently get macOS's bundled, older `psql` and the same version error.

Check what you've actually got with `pg_dump --version` and `psql --version`.

### Symptom: backup job fails with "connection refused"

The job can't reach the database. Work from the database outward:

```bash
docker ps | grep homelab-postgres        # is the container running?
docker port homelab-postgres             # is 5432 published?
```

If the container is up and the port is published, suspect a stale `host.k3d.internal` — see the [cluster-host recipe](/homelab/cluster-host). That single issue explains most "the database was fine but the cluster couldn't reach it" reports.

### Symptom: backup job fails with "Access Denied" from R2

The R2 credentials are wrong or missing. Re-sync the backup secret from the vault, and confirm the R2 endpoint URL is the right `https://<account-id>.r2.cloudflarestorage.com` form.

### Symptom: `pg_restore` prints a wall of warnings

If the warnings are about roles, ownership, or privileges — that's expected. `--no-owner` and `--no-privileges` deliberately skip restoring those, because the source and destination use different database roles. The restore still succeeded. Confirm by checking table counts (step 4 above); don't be alarmed by the noise.
