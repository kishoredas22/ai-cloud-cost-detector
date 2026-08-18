# Prompt 0: GCP Prerequisites

**This is a runbook, not a build prompt.** Nothing here produces code — these are the commands *you* run to make the project ready. Every later prompt assumes this is done; skip it and prompt 01 fails on its first `gcloud` call.

There is no Azure equivalent to this step. The Azure original needs only `az login`, because Azure Resource Manager APIs are on by default. On GCP, both the APIs and the IAM grants are explicit.

## 1. Set your project

```bash
export PROJECT_ID="your-project-id"
gcloud config set project "$PROJECT_ID"
```

## 2. Enable the required APIs

```bash
gcloud services enable \
  cloudasset.googleapis.com \
  recommender.googleapis.com \
  aiplatform.googleapis.com \
  sqladmin.googleapis.com \
  cloudresourcemanager.googleapis.com \
  --project="$PROJECT_ID"
```

| API | Needed for |
|---|---|
| `cloudasset` | `gcloud asset search-all-resources` — the resource inventory |
| `recommender` | `gcloud recommender recommendations list` — priced cost findings |
| `aiplatform` | Vertex AI Gemini |
| `sqladmin` | Cloud SQL for PostgreSQL |
| `cloudresourcemanager` | `gcloud projects list` |

Enablement is asynchronous and can take a minute or two to propagate. Confirm before moving on:

```bash
gcloud services list --enabled --project="$PROJECT_ID" \
  --filter="config.name:(cloudasset OR recommender OR aiplatform OR sqladmin)"
```

## 3. Create a service account and grant roles

```bash
export SA_NAME="cost-detective"
export SA_EMAIL="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud iam service-accounts create "$SA_NAME" \
  --display-name="AI GCP Cost Detective" \
  --project="$PROJECT_ID"

for ROLE in \
  roles/cloudasset.viewer \
  roles/recommender.viewer \
  roles/browser \
  roles/aiplatform.user \
  roles/cloudsql.client
do
  gcloud projects add-iam-policy-binding "$PROJECT_ID" \
    --member="serviceAccount:${SA_EMAIL}" \
    --role="$ROLE" \
    --condition=None
done
```

| Role | Grants |
|---|---|
| `roles/cloudasset.viewer` | `cloudasset.assets.searchAllResources` |
| `roles/recommender.viewer` | Read access across all recommenders |
| `roles/browser` | Project metadata for `gcloud projects list` |
| `roles/aiplatform.user` | Vertex AI Gemini inference |
| `roles/cloudsql.client` | Connect to the Cloud SQL instance |

`roles/recommender.viewer` is the broad grant. To tighten it, replace it with the per-product viewer roles you actually use — `roles/recommender.computeViewer` and `roles/recommender.cloudsqlViewer` cover the six recommenders in prompt 01.

**All roles here are read-only against the audited project.** The app never mutates infrastructure; it emits `gcloud` commands for a human to run. Do not grant it write roles.

## 4. Set up Application Default Credentials

The backend shells out to `gcloud` *and* calls Vertex AI, and both use ADC. This is why the stack needs no LLM API key.

For local development:

```bash
gcloud auth login
gcloud auth application-default login
gcloud auth application-default set-quota-project "$PROJECT_ID"
```

For a deployed backend, attach the service account to the runtime (Cloud Run, GKE Workload Identity, or a VM service account) so ADC resolves automatically. Avoid downloading a service-account key file if you have any alternative.

Verify:

```bash
gcloud auth application-default print-access-token >/dev/null && echo "ADC OK"
```

## 5. Create the Cloud SQL instance

```bash
gcloud sql instances create cost-detective-db \
  --database-version=POSTGRES_15 \
  --tier=db-f1-micro \
  --region=us-central1 \
  --project="$PROJECT_ID"

gcloud sql databases create costdetective \
  --instance=cost-detective-db --project="$PROJECT_ID"

gcloud sql users set-password postgres \
  --instance=cost-detective-db --prompt-for-password --project="$PROJECT_ID"
```

Note the **instance connection name** — prompt 03 needs it:

```bash
gcloud sql instances describe cost-detective-db \
  --project="$PROJECT_ID" --format="value(connectionName)"
```

There is a pleasing irony in `db-f1-micro` being the right tier here: this is a low-traffic internal tool, and a cost-auditing app that over-provisions its own database would be its own first finding.

## 6. Confirm the Gemini model ID

Prompt 02 reads the model from a `GEMINI_MODEL` environment variable rather than hardcoding it, because Vertex's Gemini catalogue moves quickly. Resolve the current Pro-tier model ID before you build:

```bash
gcloud ai models list --region=us-central1 --project="$PROJECT_ID"
```

or check **Vertex AI → Model Garden** in the Cloud Console. Pick a Pro-tier model for analysis quality; a Flash-tier model is the cost step-down if scans get frequent.

## 7. Smoke-test the three commands prompt 01 depends on

If any of these fail, fix it here — do not carry the failure into prompt 01.

```bash
# Project listing
gcloud projects list --format=json

# Resource inventory
gcloud asset search-all-resources --scope="projects/${PROJECT_ID}" --format=json --limit=5

# Cost recommendations (zonal recommender — substitute a zone you actually use)
gcloud recommender recommendations list \
  --project="$PROJECT_ID" \
  --location=us-central1-a \
  --recommender=google.compute.instance.IdleResourceRecommender \
  --format=json
```

An empty `[]` from the recommender is a success, not a failure — it means Google has no idle-VM findings in that zone. Recommenders also need roughly a week of observed usage data before they emit anything, so a freshly created project will legitimately return nothing.

---

Once all three commands return JSON, proceed to `01-fastapi-gcloud-scan.md`. Refer to `Architecture.MD` and `RequestFlow.MD` for the overall design.
