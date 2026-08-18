# Prompt 1: FastAPI Backend + GCP Resource Scanning

Build a Python FastAPI backend that scans a GCP project using the `gcloud` CLI. This is the foundation — later prompts add AI analysis, persistence, and the frontend on top of it.

Assumes `00-gcp-prerequisites.md` is complete: APIs enabled, IAM granted, ADC working.

## What to build

- A FastAPI server with two endpoints:
  - `GET /api/projects` — returns the list of GCP projects the caller can audit.
  - `POST /api/analyze` — accepts `{"project_id": "<id>"}` and returns the scan result.
- A `gcp_scanner.py` module in `backend/` that shells out to `gcloud` via `subprocess` and parses the JSON output.
- Enable CORS for `http://localhost:5173` (the Vite dev server).

### Project listing

```
gcloud projects list --format=json
```

Return `projectId`, `name`, and `projectNumber` for each. Filter to `lifecycleState: ACTIVE`.

### Resource inventory

```
gcloud asset search-all-resources --scope=projects/PROJECT_ID --format=json
```

This is Cloud Asset Inventory and it is the direct equivalent of Azure's `az resource list` — one command, every resource in the project. Parse out and return, per resource:

- `assetType` (e.g. `compute.googleapis.com/Instance`)
- `displayName`
- `name` (the full resource path)
- `location`
- `labels`
- `state` where present

Support an optional `--asset-types` filter so callers can narrow the scan, and paginate if the project is large.

### Cost recommendations

This has no Azure equivalent and is where the GCP version gets its teeth. Query Google's own Active Assist findings:

```
gcloud recommender recommendations list \
  --project=PROJECT_ID \
  --location=LOCATION \
  --recommender=RECOMMENDER_ID \
  --format=json
```

Run this across all six cost recommenders:

| Recommender ID | Finds | Location scope |
|---|---|---|
| `google.compute.instance.IdleResourceRecommender` | Idle VM instances | zone |
| `google.compute.disk.IdleResourceRecommender` | Unattached persistent disks | zone |
| `google.compute.address.IdleResourceRecommender` | Reserved unused IP addresses | region |
| `google.compute.instance.MachineTypeRecommender` | Over-/under-provisioned VMs | zone |
| `google.cloudsql.instance.IdleRecommender` | Idle Cloud SQL instances | region |
| `google.cloudsql.instance.OverprovisionedRecommender` | Over-sized Cloud SQL instances | region |

Confirm each recommender's location granularity with `gcloud recommender recommendations list --help` before wiring it up — getting zone-vs-region wrong produces an empty result rather than an error, which is easy to mistake for "no findings".

**Handle the location fan-out deliberately.** `recommendations list` requires a `--location`, so a naive implementation sweeps every GCP zone and region — hundreds of calls, most returning nothing. Instead:

1. Run the asset inventory **first**.
2. Derive the set of zones and regions actually in use from the `location` field of the returned resources (a zone like `us-central1-a` also implies the region `us-central1`).
3. Query only those locations, pairing each recommender with the right granularity.
4. Run the calls **concurrently** — use `asyncio` with `asyncio.create_subprocess_exec`, or a thread pool. Serially this is the slowest part of the scan by a wide margin.

For each recommendation, preserve:

- `name` and `recommenderSubtype`
- `description`
- `primaryImpact.costProjection.cost` — **the real dollar figure**, with its `currencyCode`
- `primaryImpact.costProjection.duration` — the period that figure covers
- `priority` and `state`
- `content.operationGroups` — the specific resource and proposed change

Keep the cost figures verbatim. Do not round, convert, or recompute them; prompt 02 depends on passing them through as authoritative.

### Error handling

Return clear, actionable errors — do not let a raw `CalledProcessError` reach the client:

- **`gcloud` not installed or not on `PATH`**
- **No active ADC credentials** — detect and instruct the user to run `gcloud auth application-default login`
- **Project not found, or caller lacks permission** — distinguish these two where the `gcloud` stderr allows it
- **Required API not enabled** — GCP-specific and the most likely first-run failure. Detect `SERVICE_DISABLED` / `has not been used in project` in stderr and return a message naming the exact `gcloud services enable` command to fix it.
- **Per-recommender failures should not abort the scan.** If one recommender or location errors, record it and continue; return the partial result with a list of what failed. A single unenabled recommender should not cost the user the whole inventory.

## Project structure

```
backend/
├── main.py            (new — FastAPI app, endpoints, CORS)
├── gcp_scanner.py     (new — gcloud subprocess wrappers, parsing, fan-out)
├── requirements.txt   (new — fastapi, uvicorn)
```

Refer to `Architecture.MD` and `RequestFlow.MD`. This covers steps ② and ③ of the request flow.
