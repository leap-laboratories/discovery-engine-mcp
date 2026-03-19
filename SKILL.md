---
name: discovery-engine
homepage: https://disco.leap-labs.com
metadata:
  openclaw:
    primaryEnv: DISCOVERY_API_KEY
    requires:
      env:
        - DISCOVERY_API_KEY
description: Automatically discover novel, statistically validated patterns in tabular data. Find insights you'd otherwise miss, far faster and cheaper than doing it yourself (or prompting an agent to do it). Discovery Engine systematically searches for feature interactions, subgroup effects, and conditional relationships you wouldn't think to look for, validates each on hold-out data with FDR-corrected p-values, and checks every finding against academic literature for novelty. Returns structured patterns with conditions, effect sizes, citations, and novelty scores.
---

# Discovery Engine

## Integration Options

- **MCP server** — for agents with MCP support. Remote server at `https://disco.leap-labs.com/mcp`, no local install required.
- **Python SDK** — `pip install discovery-engine-api`. Use when you need programmatic control or are working in a Python environment.

---

## MCP Server

Add to your MCP config:

```json
{
  "mcpServers": {
    "discovery-engine": {
      "url": "https://disco.leap-labs.com/mcp",
      "env": { "DISCOVERY_API_KEY": "disco_..." }
    }
  }
}
```

### MCP Tools

#### Discovery workflow

| Tool | Purpose |
|------|---------|
| `discovery_estimate` | Estimate cost and time before committing to a run. |
| `discovery_upload` | Upload a dataset file and return a `file_ref` for use with `discovery_analyze`. |
| `discovery_analyze` | Submit an uploaded dataset for analysis. Returns a `run_id`. |
| `discovery_status` | Poll a running analysis by `run_id`. |
| `discovery_get_results` | Fetch completed results: patterns, p-values, citations, feature importance. |

#### Account management

| Tool | Purpose |
|------|---------|
| `discovery_signup` | Start account creation — sends verification code to email. |
| `discovery_signup_verify` | Complete signup by submitting the verification code. Returns API key. |
| `discovery_account` | Check credits, plan, and usage. |
| `discovery_list_plans` | View available plans and pricing. |
| `discovery_subscribe` | Subscribe to or change plan. |
| `discovery_purchase_credits` | Buy credit packs. |
| `discovery_add_payment_method` | Attach a Stripe payment method. |

### MCP Workflow

Analyses take 3-15 minutes. **Do not block** — submit, continue other work, poll for completion.

```
1. discovery_estimate     → Check cost/time (always do this for private runs)
2. discovery_upload       → Upload the file, get file_ref
3. discovery_analyze      → Submit file_ref, get run_id
4. discovery_status       → Poll until status is "completed"
5. discovery_get_results  → Fetch patterns, summary, feature importance
```

### MCP Tool Reference

---

#### `discovery_estimate`

Estimate cost and time before committing to a run. Always call this before a private run.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file_size_mb` | float | required | Dataset size in MB |
| `num_columns` | int | required | Number of columns |
| `num_rows` | int | optional | Row count — improves time estimate accuracy |
| `depth_iterations` | int | 1 | Search depth (1=fast, higher=deeper) |
| `visibility` | string | `"public"` | `"public"` or `"private"` |
| `api_key` | string | optional | Falls back to `DISCOVERY_API_KEY` env var |

```
discovery_estimate(file_size_mb=10.5, num_columns=25, num_rows=5000, depth_iterations=2, visibility="private")
→ {
    "cost": {
      "credits": 21,
      "usd": 21.0,
      "free_alternative": true
    },
    "time_estimate": {
      "low_seconds": 180,
      "median_seconds": 360,
      "high_seconds": 720
    },
    "account": {
      "credits_available": 10,
      "sufficient": false
    }
  }
```

`free_alternative: true` means the same dataset can be run publicly at depth=1 for free (results will be published).

---

#### `discovery_upload`

Upload a dataset file. Returns a `file_ref` to pass to `discovery_analyze`. Supported formats: CSV, TSV, Excel (.xlsx), JSON, Parquet, ARFF, Feather. Max 5 GB.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file_content` | string | required | File contents, base64-encoded |
| `file_name` | string | `"data.csv"` | Filename with extension — used for format detection |
| `api_key` | string | optional | Falls back to `DISCOVERY_API_KEY` env var |

```
discovery_upload(file_content="<base64>", file_name="patients.csv")
→ {
    "file": {"key": "uploads/abc123/patients.csv", "name": "patients.csv", "size": 11010048, "fileHash": "sha256:..."},
    "columns": [
      {"name": "age", "type": "continuous", "data_type": "int"},
      {"name": "treatment", "type": "categorical", "data_type": "string"},
      {"name": "outcome", "type": "continuous", "data_type": "float"}
    ],
    "row_count": 4200
  }
```

Pass the full return value as `file_ref` to `discovery_analyze`. Inspect `columns` to confirm column names and choose `target_column`.

---

#### `discovery_analyze`

Submit an uploaded dataset for analysis. Returns immediately with a `run_id` — the analysis runs asynchronously (3-15 min). Call `discovery_upload` first to get `file_ref`.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file_ref` | object | required | Return value from `discovery_upload` |
| `target_column` | string | required | Column to analyze — what drives it? |
| `depth_iterations` | int | 1 | Search depth. Max: `num_columns - 2`. Public runs locked to 1. |
| `visibility` | string | `"public"` | `"public"` (free, results published) or `"private"` (costs credits) |
| `column_descriptions` | object | optional | Maps column names to descriptions — **always provide if column names are non-obvious** (e.g. `{"col_7": "patient age", "feat_a": "blood pressure"}`). Significantly improves pattern quality. |
| `excluded_columns` | array | optional | Column names to exclude from analysis |
| `title` | string | optional | Title for the analysis report |
| `description` | string | optional | Description of the dataset |
| `author` | string | optional | Author name |
| `source_url` | string | optional | URL of the original data source |
| `api_key` | string | optional | Falls back to `DISCOVERY_API_KEY` env var |

```
discovery_analyze(
  file_ref={"file": {...}, "columns": [...], "row_count": 4200},
  target_column="outcome",
  visibility="private",
  depth_iterations=2,
  column_descriptions={"col_7": "patient age", "feat_a": "blood pressure"},
  excluded_columns=["patient_id"],
  title="Patient outcome study"
)
→ {
    "run_id": "a1b2c3d4-e5f6-...",
    "status": "pending"
  }
```

---

#### `discovery_status`

Poll the status of a running analysis. Lightweight — does not return results. Use this while waiting for completion.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `run_id` | string | required | Run ID from `discovery_analyze` |
| `api_key` | string | optional | Falls back to `DISCOVERY_API_KEY` env var |

```
discovery_status(run_id="a1b2c3d4-e5f6-...")
→ {
    "run_id": "a1b2c3d4-e5f6-...",
    "status": "processing",    # "pending" | "processing" | "completed" | "failed"
    "job_id": "job_xyz",
    "job_status": "running",
    "error_message": null
  }
```

Poll every 30-60 seconds. Call `discovery_get_results` once `status` is `"completed"`.

---

#### `discovery_get_results`

Fetch full results of a completed run. Only call after `discovery_status` returns `"completed"`.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `run_id` | string | required | Run ID from `discovery_analyze` |
| `api_key` | string | optional | Falls back to `DISCOVERY_API_KEY` env var |

```
discovery_get_results(run_id="a1b2c3d4-e5f6-...")
→ {
    "run_id": "a1b2c3d4-...",
    "status": "completed",
    "report_url": "https://disco.leap-labs.com/reports/a1b2c3d4-...",
    "summary": {
      "overview": "14 patterns found, 5 novel...",
      "key_insights": ["Humidity × wind speed interaction increases yield by 34%", ...]
    },
    "patterns": [
      {
        "id": "p-1",
        "description": "When humidity is 72-89% AND wind speed < 12 km/h, yield increases 34%",
        "conditions": [
          {"feature": "humidity_pct", "type": "continuous", "min_value": 72.0, "max_value": 89.0},
          {"feature": "wind_speed_kmh", "type": "continuous", "min_value": 0.0, "max_value": 12.0}
        ],
        "p_value": 0.003,
        "novelty_type": "novel",
        "novelty_explanation": "This interaction has not been reported in the literature.",
        "citations": [{"title": "...", "authors": [...], "year": "2021", "doi": "..."}],
        "target_change_direction": "max",
        "abs_target_change": 0.34,
        "support_percentage": 16.9
      }
    ],
    "feature_importance": {
      "scores": [{"feature": "humidity_pct", "score": 1.82}, ...]
    },
    "hints": ["15 deeper patterns available — upgrade to explore further."]
  }
```

Always share `report_url` with the user — it links to an interactive visual report. Check `hints` for suggestions (e.g. deeper analysis available on higher plans).

---

#### `discovery_list_plans`

List available plans and pricing. No authentication required.

```
discovery_list_plans()
→ [
    {"id": "free_tier", "name": "Explorer",   "price_usd": 0,   "monthly_credits": 10},
    {"id": "tier_1",    "name": "Researcher", "price_usd": 49,  "monthly_credits": 50},
    {"id": "tier_2",    "name": "Team",       "price_usd": 199, "monthly_credits": 200}
  ]
```

Call this before `discovery_subscribe` to show the user their options.

---

#### `discovery_account`

Check account status: plan, credits, payment method. Call this before any private run to confirm sufficient credits.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `api_key` | string | optional | Falls back to `DISCOVERY_API_KEY` env var |

```
discovery_account()
→ {
    "plan": "free_tier",
    "credits": {
      "total": 8,
      "subscription": 8,
      "purchased": 0
    },
    "has_payment_method": false,
    "stripe_publishable_key": "pk_live_...",
    "stripe_customer_id": "cus_..."
  }
```

`stripe_publishable_key` is needed to tokenize a card via Stripe's API when attaching a payment method.

---

#### `discovery_signup`

Start account creation. Sends a 6-digit verification code to the email address. No authentication required.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `email` | string | required | Email address for the new account |
| `name` | string | optional | Display name (defaults to email local part) |

```
discovery_signup(email="user@example.com", name="Alice")
→ {"status": "verification_required", "email": "user@example.com"}
```

The user must check their inbox. Follow up with `discovery_signup_verify`. Returns 409 if the email is already registered.

---

#### `discovery_signup_verify`

Complete signup by submitting the verification code from email. Returns an API key on success. No authentication required.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `email` | string | required | Same email used in `discovery_signup` |
| `code` | string | required | 6-digit code from the verification email |

```
discovery_signup_verify(email="user@example.com", code="482931")
→ {
    "key": "disco_abc123...",
    "tier": "free_tier",
    "credits": 10
  }
```

Store the returned key as `DISCOVERY_API_KEY`. Free tier: 10 credits/month, unlimited public runs, no card required.

---

#### `discovery_add_payment_method`

Attach a Stripe payment method to the account. Required before purchasing credits or subscribing to a paid plan.

Card data never touches Discovery Engine — tokenize it via Stripe's API first using the `stripe_publishable_key` from `discovery_account`:

```bash
curl -X POST https://api.stripe.com/v1/payment_methods \
  -u "pk_live_...:" \
  -d "type=card" \
  -d "card[number]=4242424242424242" \
  -d "card[exp_month]=12" \
  -d "card[exp_year]=2028" \
  -d "card[cvc]=123"
# → {"id": "pm_...", "type": "card", ...}
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `payment_method_id` | string | required | Stripe payment method ID (`pm_...`) |
| `api_key` | string | optional | Falls back to `DISCOVERY_API_KEY` env var |

```
discovery_add_payment_method(payment_method_id="pm_1abc...")
→ {"payment_method_attached": true, "card_last4": "4242", "card_brand": "visa"}
```

---

#### `discovery_purchase_credits`

Buy credit packs. Credits cost $1.00 each, sold in packs of 20 ($20/pack). Requires a payment method on file.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `packs` | int | 1 | Number of 20-credit packs to purchase |
| `api_key` | string | optional | Falls back to `DISCOVERY_API_KEY` env var |

```
discovery_purchase_credits(packs=2)
→ {
    "purchased_credits": 40,
    "total_credits": 48,
    "charge_amount_usd": 40.0,
    "stripe_payment_id": "pi_..."
  }
```

---

#### `discovery_subscribe`

Subscribe to or change plan. Paid plans require a payment method on file. Credits roll over month to month on paid plans.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `plan` | string | required | `"free_tier"`, `"tier_1"`, or `"tier_2"` |
| `api_key` | string | optional | Falls back to `DISCOVERY_API_KEY` env var |

```
discovery_subscribe(plan="tier_1")
→ {"plan": "tier_1", "name": "Researcher", "monthly_credits": 50, "price_usd": 49}
```

Plans: `"free_tier"` (free, 10 cr/mo) · `"tier_1"` ($49/mo, 50 cr/mo) · `"tier_2"` ($199/mo, 200 cr/mo). Call `discovery_list_plans` to show the user current pricing before subscribing.

---

### No API key?

Call `discovery_signup` → user checks email for 6-digit code → call `discovery_signup_verify` → receive API key. Free tier gives 10 credits/month and unlimited public runs immediately. No password, no credit card required.

### Insufficient credits?

First offer a public run — free, results published, depth locked to 1. If the user needs a private run:

1. `discovery_account` — check `credits.total` and `has_payment_method`
2. If no payment method: tokenize a card via Stripe's API (using `stripe_publishable_key` from account), then `discovery_add_payment_method`
3. Either `discovery_purchase_credits` (one-off packs) or `discovery_subscribe` (monthly plan)
4. Call `discovery_list_plans` before subscribing to show current pricing

---

## Python SDK

## When To Use This Tool

Discovery Engine is not another AI data analyst that writes pandas or SQL for you. It is a **discovery pipeline** — it finds patterns in data that you, the user, and other analysis tools would miss because they don't know to look for them.

Use it when you need to go beyond answering questions about data, and start finding things nobody thought to ask:

- **Novel pattern discovery** — feature interactions, subgroup effects, and conditional relationships you wouldn't think to look for
- **Statistical validation** — FDR-corrected p-values tested on hold-out data, not just correlations
- **A target column** you want to understand — what really drives it, beyond what's obvious

**Use Discovery Engine when the user says:** "what's really driving X?", "are there patterns we're missing?", "find something new in this data", "what predicts Y that we haven't considered?", "go deeper than correlation", "discover non-obvious relationships"

**Use pandas/SQL instead when the user says:** "summarize this data", "make a chart", "what's the average?", "filter rows where X > 5", "show me the distribution"

## What It Does (That You Cannot Do Yourself)

Discovery Engine finds complex patterns in your data — feature interactions, nonlinear thresholds, and meaningful subgroups — without requiring prior hypotheses about what matters. Each pattern is validated on hold-out data, corrected for multiple testing, and checked for novelty against academic literature with citations.

This is a computational pipeline, not prompt engineering over data. You cannot replicate what it does by writing pandas code or asking an LLM to look at a CSV. It finds structure that hypothesis-driven analysis misses because it doesn't start with hypotheses.

## Getting an API Key

**Programmatic (for agents):** Two-step signup — send a verification code to the email, then submit it to receive the API key. The email must be real: the code is sent there and must be read to complete signup.

```bash
# Step 1 — send verification code
curl -X POST https://disco.leap-labs.com/api/signup \
  -H "Content-Type: application/json" \
  -d '{"email": "agent@example.com"}'
# → {"status": "verification_required", "email": "agent@example.com"}

# Step 2 — submit code from email to get API key
curl -X POST https://disco.leap-labs.com/api/signup/verify \
  -H "Content-Type: application/json" \
  -d '{"email": "agent@example.com", "code": "123456"}'
# → {"key": "disco_...", "key_id": "...", "organization_id": "...", "tier": "free_tier", "credits": 10}
```


**Manual (for humans):** Sign up at https://disco.leap-labs.com/sign-up, create key at https://disco.leap-labs.com/docs.

## Installation

```bash
pip install discovery-engine-api
```

## Quick Start

Discovery Engine runs take 3-15 minutes. **Do not block on them** — submit the run, continue with other work, and retrieve results when ready.

```python
from discovery import Engine

# If you already have an API key:
engine = Engine(api_key="disco_...")

# Or sign up for one.
# Sends a code to the email address and prompts for it interactively.
# Requires a terminal — for fully automated agents, use the two-step REST API
# in the "Getting an API Key" section above instead.
engine = await Engine.signup(email="agent@example.com")

# One-call method: submit, poll, and return results automatically
result = await engine.discover(
    file="data.csv",
    target_column="outcome",
)

# result.patterns contains the discovered patterns
for pattern in result.patterns:
    if pattern.p_value < 0.05 and pattern.novelty_type == "novel":
        print(f"{pattern.description} (p={pattern.p_value:.4f})")
```

### Inspecting Columns Before Running

If you need to see the dataset's columns before choosing a target column, upload first and inspect:

```python
# Upload once and get the server's parsed column list
upload = await engine.upload_file(file="data.csv", title="My dataset")
print(upload["columns"])   # [{"name": "col1", "type": "continuous", ...}, ...]
print(upload["rowCount"])  # e.g., 5000

# Pass the result to avoid re-uploading
result = await engine.run_async(
    file="data.csv",
    target_column="col1",
    wait=True,
    upload_result=upload,  # skips the upload step
)
```

### Running in the Background

If you need to do other work while Discovery Engine runs (recommended for agent workflows):

```python
# Submit and return immediately (wait=False is the default for run_async)
run = await engine.run_async(file="data.csv", target_column="outcome")
print(f"Submitted run {run.run_id}, continuing with other work...")

# ... do other things ...

# Check back later
result = await engine.wait_for_completion(run.run_id, timeout=1800)
```

This is the preferred pattern for agents. `engine.discover()` is a convenience wrapper that does this internally with `wait=True`.

**Non-async contexts:** use `engine.discover_sync()` — same signature as `discover()`, runs in a managed event loop.

## Example Output

Here's a truncated real response from a crop yield analysis (target column: `yield_tons_per_hectare`). This is what `engine.discover()` returns:

```python
EngineResult(
    run_id="a1b2c3d4-...",
    status="completed",
    task="regression",
    total_rows=5012,
    report_url="https://disco.leap-labs.com/reports/a1b2c3d4-...",

    summary=Summary(
        overview="Discovery Engine identified 14 statistically significant patterns in this "
                 "agricultural dataset. 5 patterns are novel — not reported in existing literature. "
                 "The strongest driver of crop yield is a previously unreported interaction between "
                 "humidity and wind speed at specific thresholds.",
        key_insights=[
            "Humidity alone is a known predictor, but the interaction with low wind speed at "
            "72-89% humidity produces a 34% yield increase — a novel finding.",
            "Soil nitrogen above 45 mg/kg shows diminishing returns when phosphorus is below "
            "12 mg/kg, contradicting standard fertilization guidelines.",
            "Planting density has a non-linear effect: the optimal range (35-42 plants/m²) is "
            "narrower than current recommendations suggest.",
        ],
        novel_patterns=PatternGroup(
            pattern_ids=["p-1", "p-2", "p-5", "p-9", "p-12"],
            explanation="5 of 14 patterns have not been reported in the agricultural literature. "
                        "The humidity × wind interaction (p-1) and the nitrogen-phosphorus "
                        "diminishing returns effect (p-2) are the most significant novel findings."
        ),
    ),

    patterns=[
        # Pattern 1: Novel multi-condition interaction
        Pattern(
            id="p-1",
            task="regression",
            target_column="yield_tons_per_hectare",
            description="When humidity is between 72-89% AND wind speed is below 12 km/h, "
                        "crop yield increases by 34% above the dataset average",
            conditions=[
                {"type": "continuous", "feature": "humidity_pct",
                 "min_value": 72.0, "max_value": 89.0, "min_q": 0.55, "max_q": 0.88},
                {"type": "continuous", "feature": "wind_speed_kmh",
                 "min_value": 0.0, "max_value": 12.0, "min_q": 0.0, "max_q": 0.41},
            ],
            p_value=0.003,                     # FDR-corrected
            p_value_raw=0.0004,
            novelty_type="novel",
            novelty_explanation="Published studies examine humidity and wind speed as independent "
                                "predictors of crop yield, but this interaction effect — where "
                                "low wind amplifies the benefit of high humidity within a specific "
                                "range — has not been reported in the literature.",
            citations=[
                {"title": "Effects of relative humidity on cereal crop productivity",
                 "authors": ["Zhang, L.", "Wang, H."], "year": "2021",
                 "journal": "Journal of Agricultural Science", "doi": "10.1017/S0021859621000..."},
                {"title": "Wind exposure and grain yield: a meta-analysis",
                 "authors": ["Patel, R.", "Singh, K."], "year": "2019",
                 "journal": "Field Crops Research", "doi": "10.1016/j.fcr.2019.03..."},
            ],
            target_change_direction="max",
            abs_target_change=0.34,
            target_score=0.81,
            support_count=847,
            support_percentage=16.9,
            target_mean=8.7,
            target_std=1.2,
        ),

        # Pattern 2: Novel — contradicts existing guidelines
        Pattern(
            id="p-2",
            task="regression",
            target_column="yield_tons_per_hectare",
            description="When soil nitrogen exceeds 45 mg/kg AND soil phosphorus is below "
                        "12 mg/kg, crop yield decreases by 18% — a diminishing returns effect "
                        "not captured by standard fertilization models",
            conditions=[
                {"type": "continuous", "feature": "soil_nitrogen_mg_kg",
                 "min_value": 45.0, "max_value": 98.0, "min_q": 0.72, "max_q": 1.0},
                {"type": "continuous", "feature": "soil_phosphorus_mg_kg",
                 "min_value": 1.0, "max_value": 12.0, "min_q": 0.0, "max_q": 0.31},
            ],
            p_value=0.008,
            p_value_raw=0.0012,
            novelty_type="novel",
            novelty_explanation="Nitrogen-phosphorus balance is studied extensively, but the "
                                "specific threshold at which high nitrogen becomes counterproductive "
                                "under low phosphorus conditions has not been quantified in field studies.",
            citations=[
                {"title": "Nitrogen-phosphorus interactions in cereal cropping systems",
                 "authors": ["Mueller, T.", "Fischer, A."], "year": "2020",
                 "journal": "Nutrient Cycling in Agroecosystems", "doi": "10.1007/s10705-020-..."},
            ],
            target_change_direction="min",
            abs_target_change=0.18,
            target_score=0.74,
            support_count=634,
            support_percentage=12.7,
            target_mean=5.3,
            target_std=1.8,
        ),

        # Pattern 3: Confirmatory — validates known finding
        Pattern(
            id="p-3",
            task="regression",
            target_column="yield_tons_per_hectare",
            description="When soil organic matter is above 3.2% AND irrigation is 'drip', "
                        "crop yield increases by 22%",
            conditions=[
                {"type": "continuous", "feature": "soil_organic_matter_pct",
                 "min_value": 3.2, "max_value": 7.1, "min_q": 0.61, "max_q": 1.0},
                {"type": "categorical", "feature": "irrigation_type",
                 "values": ["drip"]},
            ],
            p_value=0.001,
            p_value_raw=0.0001,
            novelty_type="confirmatory",
            novelty_explanation="The positive interaction between soil organic matter and drip "
                                "irrigation efficiency is well-documented in the literature.",
            citations=[
                {"title": "Drip irrigation and soil health: a systematic review",
                 "authors": ["Kumar, S.", "Patel, A."], "year": "2022",
                 "journal": "Agricultural Water Management", "doi": "10.1016/j.agwat.2022..."},
            ],
            target_change_direction="max",
            abs_target_change=0.22,
            target_score=0.69,
            support_count=1203,
            support_percentage=24.0,
            target_mean=7.9,
            target_std=1.5,
        ),

        # ... 11 more patterns omitted
    ],

    feature_importance=FeatureImportance(
        kind="global",
        baseline=6.5,          # Mean yield across the dataset
        scores=[
            FeatureImportanceScore(feature="humidity_pct", score=1.82),
            FeatureImportanceScore(feature="soil_nitrogen_mg_kg", score=1.45),
            FeatureImportanceScore(feature="soil_organic_matter_pct", score=1.21),
            FeatureImportanceScore(feature="irrigation_type", score=0.94),
            FeatureImportanceScore(feature="wind_speed_kmh", score=-0.67),
            FeatureImportanceScore(feature="planting_density_per_m2", score=0.58),
            # ... more features
        ],
    ),

    columns=[
        Column(name="yield_tons_per_hectare", type="continuous", data_type="float",
               mean=6.5, median=6.2, std=2.1, min=1.1, max=14.3),
        Column(name="humidity_pct", type="continuous", data_type="float",
               mean=65.3, median=67.0, std=18.2, min=12.0, max=99.0),
        Column(name="irrigation_type", type="categorical", data_type="string",
               approx_unique=4, mode="furrow"),
        # ... more columns
    ],
)
```

Key things to notice:
- **Patterns are combinations of conditions** (humidity AND wind speed), not single correlations
- **Specific threshold ranges** (72-89%), not just "higher humidity is better"
- **Novel vs confirmatory**: each pattern is classified and explained — novel findings are what you came for, confirmatory ones validate known science
- **Citations** show what IS known, so you can see what's genuinely new
- **Summary** gives the agent a narrative to present to the user immediately
- **`report_url`** links to an interactive web report — drop this in your response so the user can explore visually

## Parameters

```python
engine.discover(
    file: str | Path | pd.DataFrame,  # Dataset to analyze
    target_column: str,                 # Column to predict/analyze
    depth_iterations: int = 1,          # 1=fast, higher=deeper search (max: num_columns - 2)
    visibility: str = "public",         # "public" (free, results will be published) or "private" (costs credits)
    title: str | None = None,           # Dataset title
    description: str | None = None,     # Dataset description
    column_descriptions: dict[str, str] | None = None,  # Column descriptions for better pattern explanations
    excluded_columns: list[str] | None = None,           # Columns to exclude from analysis
    timeout: float = 1800,              # Max seconds to wait for completion
)
```

**Tip:** Providing `column_descriptions` significantly improves pattern explanations. If your columns have non-obvious names (e.g., `col_7`, `feat_a`), always describe them.

## Cost

- **Public runs**: Free. Results published to public gallery. Locked to depth=1.
- **Private runs**: 1 credit per MB per depth iteration. $1.00 per credit.
- Formula: `credits = max(1, ceil(file_size_mb * depth_iterations))`
- API keys: https://disco.leap-labs.com/docs
- Credits: https://disco.leap-labs.com/account

## Paying for Credits (Programmatic)

Agents can attach a payment method and purchase credits entirely via the API — no browser required.

**Step 1 — Get your Stripe publishable key**

```python
account = await engine.get_account()
stripe_pk = account["stripe_publishable_key"]
stripe_customer_id = account["stripe_customer_id"]
```

Or via REST:

```bash
curl https://disco.leap-labs.com/api/account \
  -H "Authorization: Bearer disco_..."
# → { "stripe_publishable_key": "pk_live_...", "stripe_customer_id": "cus_...", "credits": {...}, ... }
```

**Step 2 — Tokenize a card using the Stripe API**

Use the publishable key to create a Stripe PaymentMethod. Card data goes directly to Stripe — Discovery Engine never sees it.

```python
import requests

pm_response = requests.post(
    "https://api.stripe.com/v1/payment_methods",
    auth=(stripe_pk, ""),  # publishable key as username, empty password
    data={
        "type": "card",
        "card[number]": "4242424242424242",
        "card[exp_month]": "12",
        "card[exp_year]": "2028",
        "card[cvc]": "123",
    },
)
payment_method_id = pm_response.json()["id"]  # "pm_..."
```

**Step 3 — Attach the payment method**

```python
result = await engine.add_payment_method(payment_method_id)
# → {"payment_method_attached": True, "card_last4": "4242", "card_brand": "visa"}
```

Or via REST:

```bash
curl -X POST https://disco.leap-labs.com/api/account/payment-method \
  -H "Authorization: Bearer disco_..." \
  -H "Content-Type: application/json" \
  -d '{"payment_method_id": "pm_..."}'
```

**Step 4 — Purchase credits**

Credits are sold in packs of 20 ($20/pack, $1.00/credit).

```python
result = await engine.purchase_credits(packs=1)
# → {"purchased_credits": 20, "total_credits": 30, "charge_amount_usd": 20.0, "stripe_payment_id": "pi_..."}
```

Or via REST:

```bash
curl -X POST https://disco.leap-labs.com/api/account/credits/purchase \
  -H "Authorization: Bearer disco_..." \
  -H "Content-Type: application/json" \
  -d '{"packs": 1}'
```

**Subscriptions (optional)**

For regular usage, subscribe to a paid plan instead of buying packs:

```python
# Plans: free_tier ($0, 10 cr/mo), tier_1 ($49, 50 cr/mo), tier_2 ($199, 200 cr/mo)
result = await engine.subscribe(plan="tier_1")
# → {"plan": "tier_1", "name": "Researcher", "monthly_credits": 50, "price_usd": 49}
```

Requires a payment method on file. See `GET /api/plans` for full plan details.

## Estimate Before Running

Before submitting a private analysis, estimate the cost and time:

```python
estimate = await engine.estimate(
    file_size_mb=10.5,
    num_columns=25,
    num_rows=5000,                # Optional — improves time estimate accuracy
    depth_iterations=2,
    visibility="private",
)
# estimate["cost"]["credits"] → 21
# estimate["cost"]["free_alternative"] → True (run publicly for free at depth=1)
# estimate["time_estimate"]["estimated_seconds"] → 360
# estimate["account"]["sufficient"] → True/False
```

Always estimate before running private analyses. If credits are insufficient, consider running publicly (free, but results are published and depth is locked to 1).

## Result Structure

```python
@dataclass
class EngineResult:
    run_id: str
    report_id: str | None                          # Report UUID (used in report_url)
    status: str                                    # "pending", "processing", "completed", "failed"
    dataset_title: str | None                      # Title of the dataset
    dataset_description: str | None                # Description of the dataset
    total_rows: int | None
    target_column: str | None                      # Column being predicted/analyzed
    task: str | None                               # "regression", "binary_classification", "multiclass_classification"
    summary: Summary | None                        # LLM-generated insights
    patterns: list[Pattern]                        # Discovered patterns (the core output)
    columns: list[Column]                          # Feature info and statistics
    correlation_matrix: list[CorrelationEntry]     # Feature correlations
    feature_importance: FeatureImportance | None   # Global importance scores
    job_id: str | None                             # Job ID for tracking
    job_status: str | None                         # Job queue status
    error_message: str | None
    report_url: str | None                         # Shareable link to interactive web report
    hints: list[str]                               # Upgrade hints (non-empty for free-tier users with hidden patterns)
    hidden_deep_count: int                         # Patterns hidden due to depth limit (free tier)
    hidden_deep_novel_count: int                   # Novel patterns hidden due to depth limit

@dataclass
class Pattern:
    id: str
    description: str                    # Human-readable description of the pattern
    conditions: list[dict]              # Conditions defining the pattern (feature ranges/values)
    p_value: float                      # FDR-adjusted p-value (lower = more significant)
    p_value_raw: float | None           # Raw p-value before FDR adjustment
    novelty_type: str                   # "novel" (new finding) or "confirmatory" (known in literature)
    novelty_explanation: str            # Why this is novel or confirmatory
    citations: list[dict]               # Academic citations supporting novelty assessment
    target_change_direction: str        # "max" (increases target) or "min" (decreases target)
    abs_target_change: float            # Magnitude of effect
    support_count: int                  # Number of rows matching this pattern
    support_percentage: float           # Percentage of dataset
    target_score: float                 # Mean target value (regression) or class fraction (classification) in the subgroup
    task: str
    target_column: str
    target_class: str | None            # For classification tasks
    target_mean: float | None           # For regression tasks
    target_std: float | None

@dataclass
class Summary:
    overview: str                       # High-level summary
    key_insights: list[str]             # Main takeaways
    novel_patterns: PatternGroup        # Novel pattern IDs and explanation
    selected_pattern_id: str | None

@dataclass
class Column:
    id: str
    name: str
    display_name: str
    type: str                           # "continuous" or "categorical"
    data_type: str                      # "int", "float", "string", "boolean", "datetime"
    enabled: bool
    description: str | None
    mean: float | None
    median: float | None
    std: float | None
    min: float | None
    max: float | None
    iqr_min: float | None
    iqr_max: float | None
    mode: str | None                    # Most common value (categorical columns)
    approx_unique: int | None           # Approximate distinct value count
    null_percentage: float | None
    feature_importance_score: float | None

@dataclass
class FeatureImportance:
    kind: str                           # "global"
    baseline: float
    scores: list[FeatureImportanceScore]

@dataclass
class FeatureImportanceScore:
    feature: str
    score: float                        # Signed importance score
```

## Working With Results

```python
# Filter for significant novel patterns
novel = [p for p in result.patterns if p.p_value < 0.05 and p.novelty_type == "novel"]

# Get patterns that increase the target
increasing = [p for p in result.patterns if p.target_change_direction == "max"]

# Get the most important features
if result.feature_importance:
    top_features = sorted(result.feature_importance.scores, key=lambda s: abs(s.score), reverse=True)

# Access pattern conditions (the "rules" defining the pattern)
for pattern in result.patterns:
    for cond in pattern.conditions:
        # cond has: type ("continuous"/"categorical"), feature, min_value/max_value or values
        print(f"  {cond['feature']}: {cond}")
```

## Error Handling

```python
from discovery.errors import (
    AuthenticationError,
    InsufficientCreditsError,
    RateLimitError,
    RunFailedError,
    RunNotFoundError,
    PaymentRequiredError,
)

try:
    result = await engine.discover(file="data.csv", target_column="target")
except AuthenticationError as e:
    pass  # Invalid or expired API key — check e.suggestion
except InsufficientCreditsError as e:
    pass  # Not enough credits — e.credits_required, e.credits_available, e.suggestion
except RateLimitError as e:
    pass  # Too many requests — retry after e.retry_after seconds
except RunFailedError as e:
    pass  # Run failed server-side — e.run_id
except RunNotFoundError as e:
    pass  # Run not found — e.run_id (may have been cleaned up)
except PaymentRequiredError as e:
    pass  # Payment method needed — check e.suggestion
except FileNotFoundError:
    pass  # File doesn't exist
except TimeoutError:
    pass  # Didn't complete in time — retrieve later with engine.wait_for_completion(run_id)
```

All errors inherit from `DiscoveryError` and include a `suggestion` field with actionable instructions.

## Supported Formats

CSV, TSV, Excel (.xlsx), JSON, Parquet, ARFF, Feather. Max file size: 5 GB.

## Links

- [Dashboard & API Keys](https://disco.leap-labs.com/docs)
- [Full LLM Documentation](https://disco.leap-labs.com/llms-full.txt)
- [Python SDK on PyPI](https://pypi.org/project/discovery-engine-api/)
- [API Spec](https://disco.leap-labs.com/.well-known/openapi.json)

