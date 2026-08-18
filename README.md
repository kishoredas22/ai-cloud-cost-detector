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
| Database | Cloud SQL for PostgreSQL |
| Live progress | FastAPI WebSocket |

See [Architecture.MD](Architecture.MD) for the layer breakdown and [RequestFlow.MD](RequestFlow.MD) for the seven-step request lifecycle.

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
| 00 | [`00-gcp-prerequisites.md`](prompts/00-gcp-prerequisites.md) | APIs, IAM, ADC — a runbook, not code |
| 01 | [`01-fastapi-gcloud-scan.md`](prompts/01-fastapi-gcloud-scan.md) | FastAPI backend + GCP resource and recommendation scanning |
| 02 | [`02-gemini-analysis.md`](prompts/02-gemini-analysis.md) | Vertex AI Gemini cost analysis |
| 03 | [`03-cloudsql-postgres-websocket.md`](prompts/03-cloudsql-postgres-websocket.md) | Cloud SQL persistence + WebSocket progress |
| 04 | [`04-react-frontend-auth.md`](prompts/04-react-frontend-auth.md) | React frontend + JWT auth |
| 05 | [`05-integrate-frontend-backend.md`](prompts/05-integrate-frontend-backend.md) | End-to-end wiring and the report UI |
