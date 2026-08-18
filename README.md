# AI GCP Cost Detective

An AI agent that audits Google Cloud spend — it scans a GCP **project**, surfaces over-provisioned and idle resources, and returns prioritised, actionable remediation with real dollar figures attached.

This repository is a **prompt series**, not an implementation. The files in [`prompts/`](prompts/) are numbered build instructions you hand to a coding agent in order; each one builds on the last until you have a running app.

This is a GCP port of the Azure-based [AI-Cloud-Cost-Detective](https://github.com/iam-veeramalla/AI-Cloud-Cost-Detective) by Abhishek Veeramalla. The build sequence, architecture, and request flow follow the original; everything cloud-specific has been re-expressed for Google Cloud.

## What it detects

- **Idle resources** — stopped-but-billing VMs, unattached persistent disks, reserved-but-unused static IP addresses
- **Over-provisioning** — VMs whose machine type exceeds observed utilisation, over-sized Cloud SQL instances
- **Waste** — idle Cloud SQL instances, orphaned infrastructure with no owning workload
- **Missing labels and governance gaps** surfaced from the resource inventory

## Architecture at a glance

| Layer | Technology |
|---|---|
| Frontend | React + Vite + TypeScript + Tailwind CSS |
| Backend | Python FastAPI, custom JWT auth |
| Cloud access | `gcloud` CLI — Cloud Asset Inventory + Recommender API |
| AI analysis | Vertex AI Gemini via the `google-genai` SDK |
| Database | Cloud SQL for PostgreSQL, **or** Postgres on GKE — see below |
| Live progress | FastAPI WebSocket |

See [Architecture.MD](Architecture.MD) for the layer breakdown and [RequestFlow.MD](RequestFlow.MD) for the seven-step request lifecycle.

## Two deployment paths

Prompts 01, 02, 04 and 05 are identical either way. The paths diverge only at prompt 03 and rejoin at prompt 06.

| | **Path A — Managed** | **Path B — Self-hosted** |
|---|---|---|
| Database | Cloud SQL for PostgreSQL | Postgres StatefulSet in an existing GKE cluster |
| Backend | Cloud Run | Deployment in GKE |
| Auth to GCP | Service account | Workload Identity — no key files |
| Backups, PITR, SLA | Managed by Google | None, by design |
| Retention | Unbounded | 15 days, CronJob-enforced |
| **Monthly cost** | **~$10–20** | **~$4–10** |

**Path A** if you have no cluster, or the data needs backups and an SLA. **Path B** if a cluster is already running and you accept that scan history is disposable — it is regenerable by re-running a scan, which is what makes a database pod defensible.

## Before you build: this is worth measuring first

The savings this tool reports come from the **Recommender API, which is free**. Google already exposes every one of those findings through `gcloud recommender` and the Console's Active Assist dashboard, at no cost and with nothing to run. This app adds cross-project aggregation, history, prioritisation, and a stakeholder-readable report — it does not find savings you could not otherwise see.

So measure your actual addressable waste before writing code. [`prompts/00`](prompts/00-gcp-prerequisites.md) §0 shows how, in about ten minutes, and [`COSTS.md`](COSTS.md) works through what running this costs against what it realistically saves.

## Why two data sources

The Azure original sends a flat resource list to an LLM and asks it to estimate savings. GCP lets us do better:

- **Cloud Asset Inventory** (`gcloud asset search-all-resources`) gives the inventory — the direct equivalent of `az resource list`.
- **The Recommender API** (`gcloud recommender recommendations list`) gives Google's own Active Assist findings, each carrying a real `costProjection` in actual currency.

The model is therefore not guessing at savings. It receives priced findings as ground truth and spends its effort on explanation, prioritisation, and turning each finding into a `gcloud` command you can run. Anything the Recommender misses but the inventory exposes is flagged separately and clearly marked as an estimate.

## Prerequisites

- `gcloud` CLI, authenticated (`gcloud auth login` and `gcloud auth application-default login`)
- A GCP project with billing enabled
- A Cloud SQL for PostgreSQL instance
- Python 3.10+, Node.js 18+

Required APIs and IAM roles are covered in [`prompts/00-gcp-prerequisites.md`](prompts/00-gcp-prerequisites.md). **Run that one first** — every later prompt assumes it is done.

Note there is no LLM API key anywhere in this stack. Vertex AI authenticates with the same Application Default Credentials the backend already uses for `gcloud`, and inference bills to the same project the tool audits.

## Build order

| # | Prompt | Builds |
|---|---|---|
| 00 | [`00-gcp-prerequisites.md`](prompts/00-gcp-prerequisites.md) | Measure-first check, APIs, IAM, ADC — a runbook, not code |
| 01 | [`01-fastapi-gcloud-scan.md`](prompts/01-fastapi-gcloud-scan.md) | FastAPI backend + GCP resource and recommendation scanning |
| 02 | [`02-gemini-analysis.md`](prompts/02-gemini-analysis.md) | Vertex AI Gemini cost analysis |
| 03 | [`03-cloudsql-postgres-websocket.md`](prompts/03-cloudsql-postgres-websocket.md) | **Path A** — Cloud SQL persistence + WebSocket progress |
| 03 | [`03-alt-postgres-gke-websocket.md`](prompts/03-alt-postgres-gke-websocket.md) | **Path B** — Postgres on GKE, 15-day retention + WebSocket progress |
| 04 | [`04-react-frontend-auth.md`](prompts/04-react-frontend-auth.md) | React frontend + JWT auth |
| 05 | [`05-integrate-frontend-backend.md`](prompts/05-integrate-frontend-backend.md) | End-to-end wiring and the report UI |
| 06 | [`06-deploy.md`](prompts/06-deploy.md) | Deployment — both paths |

Build **either** prompt 03 or 03-alt, not both. Also: [`COSTS.md`](COSTS.md) — run cost and ROI analysis.
