# Prompt 3: Cloud SQL PostgreSQL + WebSocket Progress Tracking

Build on the existing FastAPI backend. Add Cloud SQL for PostgreSQL for storing users and analysis history, and a FastAPI WebSocket for live progress updates.

## What to build

### Database (Cloud SQL for PostgreSQL)

**Connect via the Cloud SQL Python Connector or the Cloud SQL Auth Proxy — not a raw public IP.** The connector handles IAM authentication and TLS without exposing the instance to the internet, and it is the reason prompt 00 grants `roles/cloudsql.client`. Exposing a database to `0.0.0.0/0` with a password is exactly the kind of finding this tool is built to report.

```python
from google.cloud.sql.connector import Connector
```

Store the instance connection name (`project:region:instance`) and database credentials in `.env`. Retrieve the connection name with:

```
gcloud sql instances describe INSTANCE_NAME --format="value(connectionName)"
```

Create two tables on startup:

- **`users`** — `id`, `email`, `password_hash`, `created_at`
- **`analyses`** — `id`, `user_id`, `project_id`, `resources_scanned` (int), `recommendations_found` (int), `issues_found` (int), `estimated_savings` (text), `analysis_result` (jsonb), `status`, `created_at`

Note `project_id` rather than a resource-group column — a GCP project is the analysis scope. Keep `recommendations_found` separate from `issues_found`: the former is how many priced findings Google returned, the latter is how many issues the analysis reported after deduplication and inventory review. Being able to see both is genuinely useful when judging whether the model added value or just relayed the Recommender.

After the analysis completes, store the full result in `analyses`. Add a `GET /api/history` endpoint returning past analyses for the authenticated user, newest first.

### WebSocket progress

Add a WebSocket endpoint at `ws://localhost:8000/ws/progress/{analysis_id}`.

During the `POST /api/analyze` flow, push a progress message at each stage:

- `"Listing projects..."`
- `"Scanning resources in <project_id>..."`
- `"Found <n> resources"`
- `"Fetching cost recommendations..."`
- `"Analyzing with Gemini..."`
- `"Storing results..."`
- `"Analysis complete"`

The recommendation fetch is the slowest stage because of the location fan-out from prompt 01 — emit incremental progress through it (`"Checking <n> locations..."`, then a count as results arrive) rather than leaving the UI silent on the longest step.

Push a terminal error message if any stage fails, so the frontend can stop the tracker rather than hang. Handle client disconnects mid-scan without killing the analysis — let it finish and persist, so the result is still in the user's history.

### Update `.env.example`

Add:

```
CLOUD_SQL_CONNECTION_NAME=project:region:instance
DB_USER=postgres
DB_PASSWORD=
DB_NAME=costdetective
```

## Project structure update

```
backend/
├── main.py            (updated — history endpoint, WebSocket, DB init)
├── gcp_scanner.py     (updated — emits progress callbacks)
├── ai_analyzer.py     (no change)
├── db.py              (new — connector setup, table creation, queries)
├── requirements.txt   (updated — add cloud-sql-python-connector, pg8000 or asyncpg, websockets)
├── .env.example       (updated — Cloud SQL settings)
```

Refer to `Architecture.MD` and `RequestFlow.MD`. This covers steps ④ and ⑥ of the request flow.
