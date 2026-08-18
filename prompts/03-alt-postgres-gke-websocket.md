# Prompt 3 (Alternative): Postgres on GKE + WebSocket Progress Tracking

**This is Path B — a drop-in alternative to `03-cloudsql-postgres-websocket.md`. Build one or the other, not both.**

Same schema, same endpoints, same WebSocket behaviour as prompt 03, so prompts 04 and 05 work unchanged on top of either. The difference is where Postgres lives and how long data is kept.

Use this path when you already run a GKE cluster. Postgres becomes a pod on nodes you are already paying for — roughly $1/month for the persistent disk, against $8–10/month for a Cloud SQL `db-f1-micro`. See [`../COSTS.md`](../COSTS.md) for the full comparison.

## Why running Postgres in Kubernetes is defensible here

The standard advice is to not run databases in Kubernetes, and it is good advice — for data you cannot regenerate. It does not apply to this table.

Scan history here is **disposable by design**: every row is the output of a scan you can re-run in minutes against live infrastructure that is the real source of truth. Losing the volume costs you a re-scan, not a business record. The 15-day retention below is what makes that true, so **the retention policy and the pod are a package** — build both or neither. If you later add data you cannot regenerate, revisit this decision and move to Path A.

## What to build

### Namespace

Everything lives in a dedicated namespace (`cost-detective`) with a `ResourceQuota`, so a side tool cannot disturb existing cluster workloads.

### Postgres StatefulSet

- **StatefulSet, not Deployment** — you need stable network identity and a stable volume claim. A Deployment with a PVC appears to work and then corrupts data the moment it runs two replicas or reschedules unexpectedly.
- One replica. Do not add replicas expecting HA; Postgres replication is not something a `replicas: 2` gives you.
- `volumeClaimTemplates` with a **10Gi PVC**. This is heavily oversized on purpose — 10Gi is roughly the practical floor for a GCE persistent disk, while 15 days of retained history is a few megabytes at realistic scan volumes.
- **ClusterIP Service** — headless or standard, but *not* LoadBalancer or NodePort. The database must never be reachable from outside the cluster.
- Postgres password in a **Secret**, referenced via `envFrom`/`secretKeyRef`. Never in the manifest, never in the image.
- Explicit resource **requests and limits** (start around `250m` CPU / `512Mi` memory and tune). Without a limit, a runaway query can starve neighbouring workloads; without a request, the scheduler may place it somewhere it cannot function.
- Add a readiness probe using `pg_isready` so the backend does not attempt to connect before Postgres accepts connections.

### Backend connection

Connect with plain `asyncpg` over in-cluster DNS:

```
postgresql://USER:PASSWORD@postgres.cost-detective.svc.cluster.local:5432/costdetective
```

Drop the `cloud-sql-python-connector` dependency — it is Path A only.

### Schema

Identical to prompt 03, so prompts 04 and 05 do not care which path you chose:

- **`users`** — `id`, `email`, `password_hash`, `created_at`
- **`analyses`** — `id`, `user_id`, `project_id`, `resources_scanned` (int), `recommendations_found` (int), `issues_found` (int), `estimated_savings` (text), `analysis_result` (jsonb), `status`, `created_at`

**Index `created_at`.** Both the history view and the retention job below sort and filter on it.

### Retention CronJob — 15 days, oldest first

A daily `CronJob` that deletes analyses older than 15 days:

```sql
DELETE FROM analyses WHERE created_at < NOW() - INTERVAL '15 days';
```

Time-based deletion *is* first-in-first-out — the oldest rows are always the ones that age out first, so no explicit ordering or row-counting is needed.

Implementation notes:

- Run it as a `CronJob` rather than `pg_cron` or application-level cleanup. A CronJob is visible in `kubectl get cronjobs`, its runs are auditable in job history, and it fails loudly. Cleanup buried in application code is the kind of thing that silently stops running and is noticed six months later.
- Use a `postgres` image and invoke `psql`; pull credentials from the same Secret.
- Set `successfulJobsHistoryLimit` and `failedJobsHistoryLimit` so completed job objects do not accumulate.
- Set `concurrencyPolicy: Forbid` — overlapping runs of a delete job are pointless and can contend on locks.
- At a few hundred rows per month, **autovacuum reclaims the space**; do not add table partitioning for this volume.

Do **not** delete `users` on a schedule. Retention applies to scan history only — expiring accounts would log people out of their own tool.

If you want a hard ceiling on rows regardless of age, add a second condition keeping only the N most recent per user. This is optional; the time-based rule alone satisfies the requirement.

### No backups — and that is deliberate

This path has no automated backups, no point-in-time recovery, and no SLA. That is a considered trade, not an oversight: the data expires in 15 days and is regenerable by re-running a scan.

**Write this down in your own runbook**, because the reasoning is not obvious to whoever inherits the cluster. If the table ever starts holding something you cannot regenerate, this decision must be revisited — either add a `pg_dump` CronJob writing to a GCS bucket with a lifecycle rule, or move to Path A.

### WebSocket progress

Unchanged from prompt 03. Add a WebSocket endpoint at `/ws/progress/{analysis_id}` and push a message at each stage:

- `"Listing projects..."`
- `"Scanning resources in <project_id>..."`
- `"Found <n> resources"`
- `"Fetching cost recommendations..."`
- `"Analyzing with Gemini..."`
- `"Storing results..."`
- `"Analysis complete"`

The recommendation fetch is the slowest stage because of the location fan-out from prompt 01 — emit incremental progress through it rather than leaving the UI silent on the longest step. Push a terminal error message if any stage fails so the frontend can stop the tracker. Handle client disconnects mid-scan without killing the analysis, so the result still lands in history.

### Environment

```
DB_HOST=postgres.cost-detective.svc.cluster.local
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=          # from Secret, not from .env, in-cluster
DB_NAME=costdetective
RETENTION_DAYS=15
```

## Project structure update

```
backend/
├── main.py            (updated — history endpoint, WebSocket, DB init)
├── gcp_scanner.py     (updated — emits progress callbacks)
├── ai_analyzer.py     (no change)
├── db.py              (new — asyncpg pool, table creation, queries)
├── requirements.txt   (updated — add asyncpg, websockets)
├── .env.example       (updated — DB_* settings)

k8s/
├── namespace.yaml         (new — namespace + ResourceQuota)
├── postgres-secret.yaml   (new — template; do not commit real values)
├── postgres-statefulset.yaml  (new — StatefulSet + volumeClaimTemplate)
├── postgres-service.yaml  (new — ClusterIP)
├── retention-cronjob.yaml (new — daily 15-day delete)
```

Backend Deployment, Workload Identity, and NetworkPolicy manifests come in `06-deploy.md`.

Refer to `Architecture.MD` and `RequestFlow.MD`. This covers steps ④ and ⑥ of the request flow — the same steps as prompt 03, by a different route.
