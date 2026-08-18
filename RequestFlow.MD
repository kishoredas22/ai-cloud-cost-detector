# AI GCP Cost Detective — Request Flow

Seven steps, from login to rendered report. Each prompt in [`prompts/`](prompts/) closes by naming the step or steps it implements.

```
  React                FastAPI              Google Cloud          Vertex AI        Cloud SQL
    │                     │                      │                    │                │
 ①  ├── credentials ────► │ ──────────────────── validate ─────────────────────────────►│
    │ ◄──── JWT ───────── │                      │                    │                │
    │                     │                      │                    │                │
 ②  ├── project_id ─────► │                      │                    │                │
    │                     │                      │                    │                │
 ③  │                     ├── asset inventory ──►│                    │                │
    │                     ├── recommendations ──►│                    │                │
    │                     │ ◄──── resources + priced findings ────────┤                │
    │                     │                      │                    │                │
 ④  │ ◄══ WebSocket progress ══╡ (throughout ③–⑥)                    │                │
    │                     │                      │                    │                │
 ⑤  │                     ├──────── inventory + recommendations ─────►│                │
    │                     │ ◄────────── structured JSON analysis ─────┤                │
    │                     │                      │                    │                │
 ⑥  │                     ├───────────────── store result ─────────────────────────────►│
    │                     │                      │                    │                │
 ⑦  │ ◄──── report ────── ┤                      │                    │                │
```

---

**① Authentication**

The user submits email and password through the React interface. FastAPI verifies the bcrypt password hash against the `users` table in Cloud SQL and returns a signed JWT. The frontend stores the token and attaches it as `Authorization: Bearer <token>` on every subsequent request.

*Implemented by prompt 04.*

**② Project selection**

The authenticated user requests the list of projects they can audit (`GET /api/projects`, backed by `gcloud projects list`), picks one, and submits it to `POST /api/analyze` as `{"project_id": "<id>"}`. The backend creates an analysis record and returns its ID so the frontend can open the progress WebSocket.

*Implemented by prompts 01 and 05.*

**③ Resource retrieval**

The backend invokes the `gcloud` CLI twice against the selected project:

- **Inventory** — `gcloud asset search-all-resources --scope=projects/PROJECT_ID` returns every resource: asset type, name, location, labels.
- **Cost recommendations** — `gcloud recommender recommendations list` runs across six cost recommenders. Because the Recommender API is location-scoped, the backend derives the relevant zones and regions from the inventory results and queries those, rather than sweeping every GCP location.

Both datasets are normalised into a single scan result.

*Implemented by prompt 01.*

**④ Live progress updates**

While ③–⑥ run, the backend pushes stage messages through the WebSocket at `ws://localhost:8000/ws/progress/{analysis_id}`, so the user sees the scan advance rather than watching a spinner.

*Implemented by prompt 03, wired to the UI in prompt 05.*

**⑤ AI-powered analysis**

The inventory and the recommendations go to Vertex AI Gemini together. The model returns a structured JSON analysis: a summary, a list of issues with severity levels, estimated savings, and a `gcloud` remediation command per issue. Recommender-supplied cost figures are passed through as authoritative; anything the model surfaces on its own is marked as an estimate.

*Implemented by prompt 02.*

**⑥ Persistence**

The completed analysis is written to the `analyses` table in Cloud SQL — the full result as `jsonb`, plus indexed summary columns for the history view.

*Implemented by prompt 03.*

**⑦ Report delivery**

FastAPI returns the analysis to React, which renders the summary card (resources scanned, issues found, estimated savings) and the issue list, each with resource name, category, colour-coded severity, description, and a copyable `gcloud` fix command. The same view is reachable later from the history list.

*Implemented by prompt 05.*
