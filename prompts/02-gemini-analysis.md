# Prompt 2: Vertex AI Gemini Cost Analysis

Build on the existing FastAPI backend. Add AI-powered cost analysis using Vertex AI Gemini.

## What to build

- Create an `ai_analyzer.py` module in `backend/` that:
  - Takes the scan result from `gcp_scanner.py` — **both** the resource inventory and the cost recommendations — as input.
  - Builds a prompt asking the model to analyse the project for over-provisioning, idle and unused resources, misconfiguration, and cost optimisation opportunities.
  - Calls Vertex AI Gemini and returns the structured analysis.
- Update `POST /api/analyze` to call `gcp_scanner` first, pass the result to `ai_analyzer`, and return the final analysis.
- Update `requirements.txt` — add `google-genai`, `python-dotenv`.
- Add a `.env.example`.

### Client setup

Use the **Google Gen AI SDK** (`google-genai`) — this is the current SDK for Gemini on Vertex AI, superseding `vertexai.generative_models` from `google-cloud-aiplatform`.

```python
from google import genai

client = genai.Client(
    vertexai=True,
    project=os.environ["GOOGLE_CLOUD_PROJECT"],
    location=os.environ["GOOGLE_CLOUD_LOCATION"],
)
```

`vertexai=True` is what routes the SDK at Vertex rather than the Gemini Developer API. It can also be set with the environment variables `GOOGLE_GENAI_USE_VERTEXAI=true`, `GOOGLE_CLOUD_PROJECT`, and `GOOGLE_CLOUD_LOCATION`.

**Authentication is Application Default Credentials — there is no API key in this application.** The same ADC the backend already uses to shell out to `gcloud` authenticates the Gemini call, and inference bills to the same project being audited. Do not add an API-key code path; if the client fails to authenticate, the fix is `gcloud auth application-default login`, not a key.

**Read the model ID from `GEMINI_MODEL`; do not hardcode it.** Vertex's Gemini catalogue changes often enough that a literal in the source will eventually 404. Resolve the current Pro-tier model ID from Model Garden or `gcloud ai models list` (see prompt 00 §6) and set it in `.env`. Use a Pro-tier model for analysis quality; a Flash-tier model is the cost step-down if scans become frequent.

### Structured output

Do not parse free text. Constrain the response to a schema so the backend gets valid JSON every time:

```python
from google.genai.types import GenerateContentConfig

response = client.models.generate_content(
    model=os.environ["GEMINI_MODEL"],
    contents=prompt,
    config=GenerateContentConfig(
        response_mime_type="application/json",
        response_schema=AnalysisResult,   # a Pydantic model
    ),
)
```

The SDK accepts a Pydantic model for `response_schema`, so define the result shape once as Pydantic and reuse it for validation and for the FastAPI response model.

### Analysis result shape

```
summary                  str    — a few sentences of plain-language assessment
total_estimated_savings  str    — with currency, aggregated from the findings
issues[]
  ├── resource_name      str
  ├── resource_type      str
  ├── category           str    — over_provisioned | idle | unused | misconfigured
  ├── severity           str    — high | medium | low
  ├── description        str    — what is wrong and why it costs money
  ├── estimated_savings  str    — with currency
  ├── savings_source     str    — "recommender" | "estimated"
  └── fix_command        str    — a runnable gcloud command
```

### The critical prompt instruction

The Recommender API already returns real `costProjection` figures. **The model must treat those as authoritative and must not invent, adjust, or recompute them.** State this explicitly in the system instruction. Its job on those findings is to explain, prioritise, and translate each one into a `gcloud` command — not to estimate.

Structure the prompt so the two inputs are clearly separated and differently labelled:

1. **Priced findings from the Recommender API.** Every issue derived from these carries `savings_source: "recommender"` and reproduces the Google-supplied figure verbatim, including its currency code and the duration it covers.
2. **The full resource inventory.** The model may surface additional waste visible here that the Recommender did not flag — an unlabelled orphan bucket, a VM with no auto-shutdown schedule, resources in an unexpectedly expensive region. These carry `savings_source: "estimated"` and must be presented as estimates, never mixed into the priced total as if they were measured.

Also instruct it to:

- Deduplicate — the same VM can appear in both the idle-VM and machine-type recommenders; report it once, at the higher severity.
- Rank by actual dollar impact, not by how alarming the finding sounds.
- Emit `fix_command` values that are complete and runnable, with the real project, zone, and resource name substituted in — not placeholder templates.
- Acknowledge uncertainty where it exists. An idle resource is sometimes deliberate (a DR standby, a scheduled batch worker), and the analysis is more useful when it says so than when it asserts everything is waste.

### `.env.example`

```
GOOGLE_CLOUD_PROJECT=your-project-id
GOOGLE_CLOUD_LOCATION=us-central1
GEMINI_MODEL=<resolve from Model Garden — see prompt 00 §6>
```

No API key. That is the point.

### Handling large inventories

A big project can produce an inventory that is awkward to fit in one request. If the scan exceeds a sensible size, summarise the inventory by asset type and location before sending, and pass the recommendations through in full — those are the high-value, already-priced signal and should never be truncated. Do not silently drop resources; if you summarise, say so in the analysis output so the report is honest about its coverage.

## Project structure update

```
backend/
├── main.py            (updated — /api/analyze now calls the analyzer)
├── gcp_scanner.py     (no change)
├── ai_analyzer.py     (new)
├── requirements.txt   (updated — add google-genai, python-dotenv)
├── .env.example       (new — GOOGLE_CLOUD_PROJECT, GOOGLE_CLOUD_LOCATION, GEMINI_MODEL)
```

Refer to `Architecture.MD` and `RequestFlow.MD`. This covers step ⑤ of the request flow.
