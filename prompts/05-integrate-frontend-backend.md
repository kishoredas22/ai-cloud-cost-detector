# Prompt 5: End-to-End Integration

Wire the React frontend to the FastAPI backend so the full flow works, from signup through to a rendered report.

## What to build

### Protect the API routes

Protect `/api/analyze`, `/api/history`, and `/api/projects` with JWT validation via a FastAPI dependency. Extract the user ID from the token claims and scope history queries to it.

### Analysis trigger

`POST /api/analyze` accepts the selected project and validates the JWT from the `Authorization` header. It should return the `analysis_id` **immediately** so the frontend can open the progress WebSocket, then run the scan in the background — the location fan-out from prompt 01 plus the Gemini call takes long enough that a blocking request will hit gateway timeouts on a large project.

### Live progress

After triggering the analysis, the frontend opens `ws://localhost:8000/ws/progress/{analysis_id}` and renders incoming messages in the `ProgressTracker` component as an animated step list. Close the socket on the terminal message, and handle the error case so a failed scan stops the tracker rather than leaving it spinning.

### Report page

Show a summary card with:

- Resources scanned
- Recommendations returned by Google
- Issues found after analysis
- Total estimated savings

Then each issue with: resource name, category (over-provisioned / idle / unused / misconfigured), colour-coded severity (red / yellow / green), description, estimated savings, and a copyable `gcloud` fix command.

Keep the `savings_source` distinction visible from prompt 04 — Recommender figures presented as measured, model-derived ones labelled as estimates.

Add a plain caution near the remediation commands: these are read from a point-in-time scan, and an idle-looking resource is sometimes deliberate. The user runs the commands; the app never does. That boundary is the reason prompt 00 grants only read-only roles.

### History

`GET /api/history` returns past analyses for the authenticated user. Clicking an entry opens the full stored report — read it from the `analysis_result` jsonb column rather than re-running the scan.

## Test end-to-end

Confirm the whole flow: **signup → login → select project → run analysis → see live progress → view report → check history**.

Worth verifying specifically:

- A project with **no** findings renders a clean empty state rather than an error. An empty `[]` from a recommender is a valid result, and a new project will legitimately have nothing — the recommenders need roughly a week of usage data before they emit anything.
- A project the user lacks permission on returns the clear error from prompt 01, not a stack trace.
- A `fix_command` copied from the report actually runs.

## Optional extensions

Two GCP cost signals are deliberately **not** in this build. Both are worth adding if the tool gets real use:

**Billing BigQuery export.** Enabling the detailed billing export gives per-SKU, per-resource historical spend in BigQuery. That turns "estimated savings" into measured spend for *every* finding, not just the ones the Recommender priced, and lets the report show trend lines rather than a point-in-time snapshot. It requires the export to be configured in advance and takes 24–48 hours to start populating, which is why it is not in the core build.

**Per-service `gcloud` enumeration.** Cloud Asset Inventory returns resource identity and metadata but not full configuration — it will tell you a VM exists in `us-central1-a`, not that it is an `n2-standard-16` with a 2 TB SSD. Targeted calls to `gcloud compute instances list`, `gcloud compute disks list`, `gcloud sql instances list`, and `gcloud container clusters list` fill that in and let the model reason about right-sizing beyond what the Recommender already flags. The cost is a slower scan and more API surface to handle.

## Project structure update

```
backend/
├── main.py            (updated — JWT dependency on protected routes, background scan)
├── auth.py            (updated — token verification dependency)

frontend/src/
├── pages/
│   ├── Dashboard.tsx  (updated — triggers analysis, opens WebSocket)
│   ├── Report.tsx     (updated — summary card, issue list, copy buttons)
│   └── History.tsx    (updated — fetches and links to stored reports)
└── components/
    └── ProgressTracker.tsx  (updated — consumes WebSocket messages)
```

Refer to `Architecture.MD` and `RequestFlow.MD`. This covers the complete seven-step request flow, ① through ⑦.
