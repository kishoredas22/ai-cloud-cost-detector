# AI GCP Cost Detective — Architecture

## Overview

Four layers. The React frontend talks only to FastAPI; FastAPI is the only component that touches Google Cloud, Vertex AI, or the database.

```
┌───────────────────────────────────────────────────────────────┐
│  PRESENTATION LAYER                                           │
│  React + Vite + TypeScript + Tailwind (dark theme)            │
│  Login / Signup · Dashboard · ProgressTracker · Report        │
│  · History                                                    │
└───────────────────────────┬───────────────────────────────────┘
                            │ REST (JWT bearer) + WebSocket
┌───────────────────────────▼───────────────────────────────────┐
│  AUTH & API LAYER                                             │
│  Python FastAPI · Custom JWT auth (PyJWT + bcrypt)            │
│  Orchestrates the scan → analyse → persist pipeline           │
│  Pushes live progress over a WebSocket                        │
└──────┬──────────────────┬──────────────────┬──────────────────┘
       │                  │                  │
┌──────▼────────┐  ┌──────▼────────┐  ┌──────▼──────────────────┐
│ INTEGRATION   │  │ AI ANALYSIS   │  │ REAL-TIME               │
│ gcloud CLI    │  │ Vertex AI     │  │ FastAPI WebSocket       │
│ · Asset       │  │ Gemini        │  │ /ws/progress/{id}       │
│   Inventory   │  │ google-genai  │  │ Stage-by-stage updates  │
│ · Recommender │  │ SDK, ADC auth │  │                         │
│   API         │  │ Structured    │  │                         │
│               │  │ JSON output   │  │                         │
└──────┬────────┘  └───────────────┘  └─────────────────────────┘
       │
┌──────▼────────────────────────────────────────────────────────┐
│  DATA & STORAGE LAYER                                         │
│  Cloud SQL for PostgreSQL — users, analyses (jsonb results)   │
│  GCP Projects — the infrastructure under audit                │
└───────────────────────────────────────────────────────────────┘
```

## Layers

### Presentation layer

A React single-page app. Handles login and signup, lets the user pick a GCP project to audit, renders a live progress tracker while the scan runs, and displays the final report — cost breakdown, severity-ranked issues, estimated savings, and copyable `gcloud` remediation commands. A history view lists past analyses.

### Auth & API layer

Python FastAPI. Implements custom JWT authentication with credentials stored in Cloud SQL, and orchestrates the whole pipeline: resolve the project, scan it, call Gemini, persist the result, and return the report. It also owns the WebSocket connection that streams progress back to the browser while the scan is in flight.

### Integration & processing layer

Three integrations, all initiated from the backend:

- **`gcloud` CLI — Cloud Asset Inventory.** `gcloud asset search-all-resources --scope=projects/PROJECT_ID` returns a flat inventory of every resource in the project: asset type, name, location, and labels.
- **`gcloud` CLI — Recommender API.** `gcloud recommender recommendations list` returns Google's Active Assist cost findings across six recommenders (idle VM, idle disk, idle IP address, VM machine-type right-sizing, idle Cloud SQL, over-provisioned Cloud SQL). Each recommendation carries a `costProjection` with a real currency amount.
- **Vertex AI Gemini.** The inventory and the recommendations are sent together to Gemini via the `google-genai` SDK, which returns a structured JSON analysis. Authentication is Application Default Credentials — the same credentials `gcloud` uses. There is no API key.

### Data & storage layer

**Cloud SQL for PostgreSQL** holds user accounts and analysis history, with the full analysis result stored as `jsonb`. **GCP projects** are the infrastructure being audited — the app reads them, never writes to them.

## Data flow

The user authenticates through React against FastAPI, which validates credentials against Cloud SQL and issues a JWT. The user selects a project; the backend enumerates its resources through Cloud Asset Inventory and fetches cost recommendations through the Recommender API, streaming progress to the frontend over the WebSocket at each stage. Both datasets go to Vertex AI Gemini, which returns a structured analysis. The result is persisted to Cloud SQL and rendered in the React report view — cost breakdown, prioritised recommendations, and `gcloud` commands the user can copy and run.

## Deployment options

The diagram above is **Path A**, the managed topology. A second path self-hosts the data layer in an existing GKE cluster. The application code, request flow, schema, and API surface are identical — only where Postgres and the backend run differs.

| | **Path A — Managed** | **Path B — Self-hosted** |
|---|---|---|
| Database | Cloud SQL for PostgreSQL | Postgres StatefulSet in GKE |
| Backend | Cloud Run | Deployment in GKE |
| Auth to GCP | Service account | Workload Identity — no key files |
| Backups, PITR, SLA | Managed by Google | None, by design |
| Retention | Unbounded | 15 days, CronJob-enforced |
| Monthly cost | ~$10–20 | ~$4–10 |
| Build with | `prompts/03-cloudsql-postgres-websocket.md` | `prompts/03-alt-postgres-gke-websocket.md` |

Path B's storage layer:

```
┌───────────────────────────────────────────────────────────────┐
│  GKE cluster — namespace: cost-detective                      │
│                                                               │
│  ┌─────────────┐        ┌──────────────────────────────────┐  │
│  │ backend     │───────►│ postgres  StatefulSet + 10Gi PVC │  │
│  │ Deployment  │  5432  │ ClusterIP — no external exposure │  │
│  │ (Workload   │        └──────────────▲───────────────────┘  │
│  │  Identity)  │                       │                      │
│  └──────┬──────┘        ┌──────────────┴───────────────────┐  │
│         │               │ retention  CronJob, daily         │  │
│         │               │ DELETE WHERE created_at < 15d     │  │
│         │               └──────────────────────────────────┘  │
│         │      NetworkPolicy: only backend may reach :5432    │
└─────────┼─────────────────────────────────────────────────────┘
          │ ADC via Workload Identity
          ▼
   gcloud CLI · Recommender API · Vertex AI Gemini
```

**Why a database pod is defensible here.** The usual objection to running databases in Kubernetes concerns durability of irreplaceable data. It does not apply to this table: every row is the output of a scan that can be re-run in minutes against live infrastructure, which is the real source of truth. Losing the volume costs a re-scan, not a business record. The 15-day retention is what makes that true, so the retention policy and the pod are a package — one without the other is a mistake.

**Workload Identity removes key files from the design.** On Path B the backend pod gets ADC by binding its Kubernetes service account to a Google service account, so nothing needs a downloaded JSON key — for either the `gcloud` calls or Vertex AI. That is a security improvement over Path A, not only a cost one.

See [`COSTS.md`](COSTS.md) for the full comparison.

## Design notes

**A single credential.** The backend needs Application Default Credentials to shell out to `gcloud`. Vertex AI accepts those same credentials, so the entire stack authenticates with one service account and no external API keys. Nothing needs to go into a secrets manager except the database password and the JWT signing secret.

**Priced findings, not guessed ones.** Because the Recommender API returns real cost projections, the model is not asked to estimate savings for findings that already carry a figure. This is the substantive difference from a naive "send the resource list to an LLM" design.

**Read-only against the audited project.** The app never mutates the infrastructure it scans. Remediation is presented as `gcloud` commands for a human to review and run — a deliberate boundary, since idle-looking resources are sometimes deliberate.

## Key technologies

React, Vite, TypeScript, Tailwind CSS, Python FastAPI, PyJWT, bcrypt, WebSockets, `gcloud` CLI, Cloud Asset Inventory, Recommender API, Vertex AI Gemini, `google-genai`, Cloud SQL for PostgreSQL.
