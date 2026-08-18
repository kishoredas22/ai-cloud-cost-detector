# Prompt 6: Deployment

Deploy the app. Two paths, matching the choice made at prompt 03 — build the one you picked.

| | **Path A — Managed** | **Path B — Self-hosted** |
|---|---|---|
| From prompt | `03-cloudsql-postgres-websocket.md` | `03-alt-postgres-gke-websocket.md` |
| Database | Cloud SQL | Postgres StatefulSet in GKE |
| Backend | Cloud Run | Deployment in GKE |
| Auth to GCP | Service account | Workload Identity |

## Applies to both paths

### `gcloud` must be inside the backend image

`gcp_scanner.py` shells out to the `gcloud` CLI. A stock `python:3.11-slim` base image does not have it, and nothing fails at build time — the container starts fine, serves requests fine, and then every scan fails at runtime with "gcloud: not found".

Install the Google Cloud CLI in the image, or use a base image that includes it. Verify in the Dockerfile build, not after deploying:

```dockerfile
RUN gcloud --version
```

### Do not bake credentials into the image

No service-account key files, no `.env` with real values, no `GOOGLE_APPLICATION_CREDENTIALS` pointing at a copied-in JSON key. Both paths below have a keyless mechanism; use it.

### Frontend

The React app builds to static files. Serve from Cloud Storage with a load balancer, or Firebase Hosting. Point it at the backend URL via a build-time environment variable, and update the backend's CORS origin from `http://localhost:5173` to the deployed frontend origin.

---

## Path A — Cloud Run + Cloud SQL

### Build and push

Build the backend image and push to Artifact Registry.

### Deploy

Deploy to Cloud Run with:

- The **Cloud SQL connection** attached (`--add-cloudsql-instances`), so the connector reaches the instance over the built-in socket rather than a public IP.
- A **dedicated service account** carrying the roles from prompt 00 — Cloud Run gets ADC from the attached service account automatically, so no key file is needed.
- Database password and `JWT_SECRET` from **Secret Manager**, mounted as environment variables. Not as plain `--set-env-vars`.
- `--min-instances=0` so it scales to zero between scans. This is a tool used a few times a week; paying for a warm instance defeats the point.

### Watch the request timeout

A full scan — location fan-out plus the Gemini call — can exceed Cloud Run's default request timeout on a large project. Prompt 05 already returns `analysis_id` immediately and runs the scan in the background, which is what makes this survivable. Confirm that is actually how it was built before deploying, and raise `--timeout` if needed.

### WebSocket support

Cloud Run supports WebSockets, but connections are bounded by the request timeout and are not sticky across instances. With `min-instances=0` and a single concurrent user this is fine. If progress updates start dropping under real use, that is the cause.

---

## Path B — All on GKE

### Namespace and quota

Already created in prompt 03-alt. The backend joins the same `cost-detective` namespace.

### Workload Identity — the keyless auth mechanism

This is the piece that makes Path B secure rather than merely cheap. The backend pod needs ADC for two things: shelling out to `gcloud`, and calling Vertex AI. Workload Identity binds a Kubernetes service account to a Google service account so the pod gets credentials automatically, with **no key file anywhere**.

Three steps, all of which must be done or ADC silently falls back and fails:

1. The **cluster** must have Workload Identity enabled (checked in prompt 00).
2. Bind the Google service account to the Kubernetes service account with an IAM policy binding on `roles/iam.workloadIdentityUser`, for the member `serviceAccount:PROJECT.svc.id.goog[cost-detective/KSA_NAME]`.
3. **Annotate the KSA** with `iam.gke.io/gcp-service-account: GSA_EMAIL`.

Missing the annotation is the classic failure — everything applies cleanly and the pod then authenticates as the node's default service account, which usually has different permissions. Symptoms are permission errors on `gcloud`, not auth errors, which sends you debugging the wrong thing.

### Backend Deployment

- Deployment plus ClusterIP Service, using the annotated KSA.
- Resource requests and limits set explicitly.
- Readiness and liveness probes on a health endpoint.
- Config from a ConfigMap, secrets (`JWT_SECRET`, DB password) from a Secret.
- `DB_HOST` pointing at the in-cluster Postgres service DNS.

### NetworkPolicy

Restrict ingress to the Postgres pod on port 5432 to the backend pod only, selected by label.

**Verify it actually works.** A NetworkPolicy is silently inert if the cluster's network plugin does not enforce policies — the object applies successfully and does nothing. Test by exec-ing into a pod in another namespace and attempting a connection; it should fail. If it succeeds, network policy enforcement is not enabled on your cluster and Postgres is reachable cluster-wide.

### Exposing the backend

Ingress or Gateway with a managed certificate. Expose **only** the backend — Postgres stays ClusterIP and gets no ingress, no LoadBalancer, no NodePort.

## Verify the deployment

Run these after deploying, in order — each catches a failure the previous one cannot:

```bash
# Path B: confirm the pod's identity is the GSA, not the node default
kubectl exec -n cost-detective deploy/backend -- gcloud auth list

# Both paths: confirm gcloud is present and can actually scan
kubectl exec -n cost-detective deploy/backend -- \
  gcloud asset search-all-resources --scope=projects/PROJECT_ID --limit=1

# Path B: confirm Postgres is NOT reachable from outside the namespace
kubectl run probe --rm -it --image=postgres:16 -n default -- \
  psql -h postgres.cost-detective.svc.cluster.local -U postgres   # must fail

# Path B: confirm the retention job runs and deletes only aged rows
kubectl create job --from=cronjob/retention retention-manual -n cost-detective
kubectl logs -n cost-detective job/retention-manual
```

Then the real test: run one full analysis end to end through the deployed frontend and confirm the report renders with live progress.

Refer to `Architecture.MD` for both deployment topologies and [`../COSTS.md`](../COSTS.md) for what each path costs to run.
