# What This Costs, and What It Saves

Worked against a **~$3,000/month** GCP estate spread across ~10 projects. Substitute your own figures — the method matters more than the numbers.

> **Every figure below marked "estimate" is one.** Token volumes are projections from expected `search-all-resources` payload sizes and should be validated against a real scan. Per-token rates are Gemini 2.5-era figures from third-party trackers and should be reconfirmed against Google's [Vertex AI pricing page](https://cloud.google.com/vertex-ai/generative-ai/pricing) for whichever model you select. A cost tool whose own cost estimate is stale is not a good look — correct this file once you have real billing data.

## Read this first

**The savings come from the Recommender API, which is free.** [Active Assist is free](https://cloud.google.com/recommender/pricing) for all customers; only Firewall Insights is a paid premium recommender. Google already exposes every finding this tool reports through `gcloud recommender` and the Console's Active Assist dashboard, at no cost and with nothing to build.

So this app does not discover savings. It adds cross-project aggregation, history and trending, LLM prioritisation, and a report a non-engineer can act on. Whether that packaging justifies the build is the actual question, and **prompt 00 §0 tells you how to measure the answer** before writing any code.

## What it costs to run

| Line item | Path A (Cloud SQL + Cloud Run) | Path B (all on GKE) | Basis |
|---|---|---|---|
| Database | **$8–10** | **~$1** | A: `db-f1-micro`, 10 GB SSD, no HA, `us-central1`. B: 10Gi PVC on existing nodes. |
| Backend compute | **$0–3** | **~$0** | A: Cloud Run, scales to zero. B: pod on nodes already paid for. |
| Static frontend | **~$0** | **~$0** | Cloud Storage / Firebase free tier. |
| Vertex AI inference | **$2–7** | **$2–7** | See calculation below. |
| Cloud Asset Inventory | **$0** | **$0** | No charge for `search-all-resources`. |
| Recommender API | **$0** | **$0** | Free. |
| **Total** | **≈ $10–20/mo** | **≈ $4–10/mo** | |

**The AI is not the expensive part.** On Path A the always-on database costs more than the model that does the actual analysis. Path B removes that fixed floor, which inverts the bill — inference becomes dominant, so cost scales with how often you scan rather than sitting there whether you use it or not.

Path B's saving is real but conditional: it assumes **spare capacity on existing nodes**. If adding these pods forces a node scale-up, that new node costs considerably more than the Cloud SQL instance you removed, and Path A becomes the cheaper option. Check `kubectl top nodes` before committing (prompt 00 §5, Path B).

### Inference calculation

For 10 projects scanned **weekly** (40 scans/month):

| | Per scan | Per month |
|---|---|---|
| Input | ~75K tokens | **3M tokens** |
| Output | ~7K tokens | **280K tokens** |

Input is roughly 250 resources × ~150 tokens of JSON each, plus recommendations and the system prompt. Output is the structured analysis — 20–40 issues at ~150 tokens each.

| Model tier | Rate (in / out per 1M) | Monthly |
|---|---|---|
| Gemini Pro-tier | ~$1.25 / ~$10.00 | **≈ $6.55** |
| Gemini Flash-Lite | $0.10 / $0.40 | **≈ $0.41** |

Scanning **monthly instead of weekly** cuts this to under $2 on Pro-tier. Note that prompts over 200K tokens are billed at a higher long-context rate — a single very large project could cross that threshold, which is one reason prompt 02 summarises oversized inventories rather than sending everything.

Inference would have to be off by more than 10× to rival the infrastructure on Path A. Precision here is not what decides the question.

## What it could save

Against **$3,000/month**, on *identified* waste:

| Estate condition | Addressable | On $3,000/mo |
|---|---|---|
| Actively managed already | 2–5% | $60–150 |
| Never systematically audited | 10–20% | $300–600 |
| Dev/test sprawl, unowned resources | 25–30%+ | $750–900 |

These are planning ranges, not predictions. **Prompt 00 §0 replaces all of them with a measured number in about ten minutes.** Use that instead.

Three things reduce the figure that actually lands in your bill:

**Identified is not realised.** Some idle resources are deliberate — DR standbys, seasonal batch workers, warm spares held for a reason nobody wrote down. Rightsizing needs downtime windows and owner sign-off. A 30–60% realisation rate on identified waste is a fair assumption, and the bottleneck is follow-through, not detection.

**Savings are front-loaded.** The first audit takes the easy wins. Month two finds substantially less, and the curve flattens fast. Do not extrapolate month one across a year.

**Recommenders need history.** They require roughly a week of observed usage before emitting anything, so a freshly created project legitimately returns nothing. An empty result is not a bug.

## The verdict

At $4–20/month depending on path, even **$100/month realised** is a 5–25× return with a payback period under a month. The economics are not close.

But they are conditional on one thing: **someone has to act on the findings.** A report nobody reads is a monthly expense that produces reports — which, on this tool's own terms, is a finding.

The honest framing:

- **Build it** if the prompt-00 measurement shows meaningful addressable waste spread across enough projects that reviewing them individually in the Console is tedious, and there is someone who will action the output.
- **Skip it** if the measured number is small, or if nobody owns remediation. Use the free Console dashboard and spend the effort elsewhere.
- **Build it anyway** if the goal is learning the GCP cost APIs and the agent pattern — a perfectly good reason, just not an ROI one, and worth being clear-eyed about which reason applies.

## Sources

- [Recommender pricing](https://cloud.google.com/recommender/pricing) — Active Assist is free
- [Vertex AI generative AI pricing](https://cloud.google.com/vertex-ai/generative-ai/pricing) — authoritative for token rates
- [Cloud SQL pricing](https://cloud.google.com/sql/pricing) — use the calculator for your region and config
- [Understanding how cost savings are calculated](https://cloud.google.com/recommender/docs/understand-cost-recs) — how Google derives `costProjection`
