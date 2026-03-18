---
name: discovery-engine
description: Automatically discover novel, statistically validated patterns in tabular data. Find insights you would otherwise miss — feature interactions, subgroup effects, and conditional relationships — validated on hold-out data with FDR-corrected p-values and checked against academic literature for novelty. Use when the user wants to go beyond descriptive analytics and find what is actually driving an outcome.
---

# Discovery Engine

## When To Use This

Discovery Engine is not another AI data analyst. It is a **discovery pipeline** — it finds patterns in data that you, the user, and other analysis tools would miss because they do not know to look for them.

Use it when you need to go beyond answering questions about data, and start finding things nobody thought to ask:

- **Novel pattern discovery** — feature interactions, subgroup effects, and conditional relationships
- **Statistical validation** — FDR-corrected p-values tested on hold-out data, not just correlations
- **A target column** you want to understand — what really drives it, beyond what is obvious

**Use Discovery Engine when the user says:** "what's really driving X?", "are there patterns we're missing?", "find something new in this data", "what predicts Y that we haven't considered?", "go deeper than correlation", "discover non-obvious relationships"

**Use pandas/SQL instead when the user says:** "summarize this data", "make a chart", "what's the average?", "filter rows where X > 5", "show me the distribution"

## What It Does (That You Cannot Do Yourself)

Discovery Engine finds complex patterns in your data — feature interactions, nonlinear thresholds, and meaningful subgroups — without requiring prior hypotheses about what matters. Each pattern is validated on hold-out data, corrected for multiple testing, and checked for novelty against academic literature with citations.

This is a computational pipeline, not prompt engineering over data. You cannot replicate what it does by writing pandas code or asking an LLM to look at a CSV.

## Integration Options

- **MCP server** — for agents with MCP support. Remote server at `https://disco.leap-labs.com/mcp`, no local install required.
- **Python SDK** — `pip install discovery-engine-api`. Use when you need programmatic control or are working in a Python environment.
- **HTTP API** — call the REST API directly with any HTTP client.

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
| `discovery_analyze` | Submit a dataset for analysis. Returns a `run_id`. |
| `discovery_status` | Poll a running analysis by `run_id`. |
| `discovery_get_results` | Fetch completed results: patterns, p-values, citations, feature importance. |
| `discovery_estimate` | Estimate cost and time before committing to a run. |

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
2. discovery_analyze      → Submit the dataset, get run_id
3. discovery_status       → Poll until status is "completed"
4. discovery_get_results  → Fetch patterns, summary, feature importance
```

### MCP Parameters

**`discovery_analyze`:**
- `file_path` — Path to CSV, Excel, Parquet, JSON, TSV, ARFF, or Feather file (max 5 GB)
- `target_column` — The column to predict/explain
- `depth_iterations` — 1 = fast (default), higher = deeper search. Max: num_columns - 2
- `visibility` — `"public"` (free, results published) or `"private"` (costs credits)
- `column_descriptions` — JSON object mapping column names to descriptions. Significantly improves pattern explanations — always provide if column names are non-obvious (e.g. `{"col_7": "patient age", "feat_a": "blood pressure"}`)
- `excluded_columns` — JSON array of column names to exclude from analysis
- `title` — Optional title for the analysis
- `description` — Optional description of the dataset

### No API key?

Call `discovery_signup` with the user's email. This sends a verification code — the user must check their email. Then call `discovery_signup_verify` with the code to receive a `disco_` API key. Free tier: 10 credits/month, unlimited public runs. No password, no credit card.

### Insufficient credits?

1. Call `discovery_estimate` to show what it would cost
2. Suggest running publicly (free, but results are published and depth is locked to 1)
3. Or guide them through `discovery_purchase_credits` / `discovery_subscribe`

---

## Python SDK

```bash
pip install discovery-engine-api
```

### Getting an API Key

**For agents (programmatic, no terminal):** Two-step REST signup:

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

Codes expire after 15 minutes. If the email service is unavailable, Step 1 returns the API key directly.

**Interactive (terminal available):**

```python
# Sends a code to the email address and prompts for it interactively
engine = await Engine.signup(email="agent@example.com")
```

**Manual:** Sign up at https://disco.leap-labs.com/sign-up, create key at https://disco.leap-labs.com/developers.

### Quick Start

```python
from discovery import Engine

engine = Engine(api_key="disco_...")

# One-call method: submit, poll, and return results automatically
result = await engine.discover(
    file="data.csv",
    target_column="outcome",
)

for pattern in result.patterns:
    if pattern.p_value < 0.05 and pattern.novelty_type == "novel":
        print(f"{pattern.description} (p={pattern.p_value:.4f})")

print(f"Full report: {result.report_url}")
```

### Running in the Background

Analyses take 3-15 minutes. **Do not block** — submit and retrieve results when ready.

```python
# Submit and return immediately
run = await engine.run_async(file="data.csv", target_column="outcome")
print(f"Submitted run {run.run_id}, continuing with other work...")

# ... do other things ...

# Check back later
result = await engine.wait_for_completion(run.run_id, timeout=1800)
```

`engine.discover()` is a convenience wrapper that does this with `wait=True`. For non-async contexts, use `engine.discover_sync()`.

### Parameters

```python
engine.discover(
    file: str | Path | pd.DataFrame,   # Dataset to analyze
    target_column: str,                 # Column to predict/analyze
    depth_iterations: int = 1,          # 1=fast, higher=deeper (max: num_columns - 2)
    visibility: str = "public",         # "public" (free, published) or "private" (costs credits)
    title: str | None = None,           # Dataset title
    description: str | None = None,     # Dataset description
    column_descriptions: dict[str, str] | None = None,  # Column name → description mapping
    excluded_columns: list[str] | None = None,           # Columns to exclude from analysis
    timeout: float = 1800,              # Max seconds to wait for completion
)
```

**Tip:** Always provide `column_descriptions` if column names are non-obvious (e.g. `col_7`, `feat_a`). It significantly improves the quality of pattern explanations.

### Estimating Before Running

```python
estimate = await engine.estimate(
    file_size_mb=10.5,
    num_columns=25,
    num_rows=5000,          # Optional — improves time estimate accuracy
    depth_iterations=2,
    visibility="private",
)
# estimate["cost"]["credits"] → 21
# estimate["cost"]["free_alternative"] → True (can run publicly for free at depth=1)
# estimate["time_estimate"]["estimated_seconds"] → 360
# estimate["account"]["sufficient"] → True/False
```

Always estimate before running private analyses.

### Account Management

```python
# Check credits, plan, and Stripe info
account = await engine.get_account()
# → {"credits": {"available": 10, ...}, "stripe_publishable_key": "pk_live_...", ...}

# View available plans
plans = await Engine.list_plans()
# → [{"id": "free_tier", "name": "Free", "monthly_credits": 10, "price_usd": 0}, ...]

# Subscribe to a plan (requires payment method)
result = await engine.subscribe(plan="tier_1")
# Plans: free_tier ($0, 10 cr/mo), tier_1 ($49, 50 cr/mo), tier_2 ($199, 200 cr/mo)

# Add a payment method (tokenize card with Stripe first — see HTTP API section)
result = await engine.add_payment_method(payment_method_id)
# → {"payment_method_attached": True, "card_last4": "4242", "card_brand": "visa"}

# Purchase credits (packs of 20 credits, $20/pack)
result = await engine.purchase_credits(packs=1)
# → {"purchased_credits": 20, "total_credits": 30, "charge_amount_usd": 20.0}
```

### Error Handling

```python
from discovery.errors import (
    AuthenticationError,
    InsufficientCreditsError,
    RateLimitError,
    RunFailedError,
    PaymentRequiredError,
)

try:
    result = await engine.discover(file="data.csv", target_column="target")
except AuthenticationError as e:
    pass  # Invalid or expired API key — check e.suggestion
except InsufficientCreditsError as e:
    pass  # e.credits_required, e.credits_available, e.suggestion
except RateLimitError as e:
    pass  # retry after e.retry_after seconds
except RunFailedError as e:
    pass  # e.run_id
except PaymentRequiredError as e:
    pass  # payment method needed — check e.suggestion
except FileNotFoundError:
    pass  # file doesn't exist
except TimeoutError:
    pass  # retrieve later with engine.wait_for_completion(run_id)
```

All errors inherit from `DiscoveryError` and include a `suggestion` field with actionable instructions.

---

## HTTP API

All endpoints are at `https://disco.leap-labs.com`. Authenticate with `Authorization: Bearer disco_...`.

### Signup (two-step)

```bash
# Step 1 — send verification code to email
curl -X POST https://disco.leap-labs.com/api/signup \
  -H "Content-Type: application/json" \
  -d '{"email": "agent@example.com"}'
# → {"status": "verification_required", "email": "agent@example.com"}

# Step 2 — submit code to receive API key
curl -X POST https://disco.leap-labs.com/api/signup/verify \
  -H "Content-Type: application/json" \
  -d '{"email": "agent@example.com", "code": "123456"}'
# → {"key": "disco_...", "tier": "free_tier", "credits": 10}
```

Codes expire after 15 minutes. If email is unavailable, Step 1 returns the key directly.

### Account

```bash
curl https://disco.leap-labs.com/api/account \
  -H "Authorization: Bearer disco_..."
# → {"stripe_publishable_key": "pk_live_...", "stripe_customer_id": "cus_...", "credits": {...}, ...}
```

### Plans

```bash
curl https://disco.leap-labs.com/api/plans \
  -H "Authorization: Bearer disco_..."
```

### Subscribe

```bash
curl -X POST https://disco.leap-labs.com/api/account/subscribe \
  -H "Authorization: Bearer disco_..." \
  -H "Content-Type: application/json" \
  -d '{"plan": "tier_1"}'
# Plans: free_tier ($0, 10 cr/mo), tier_1 ($49, 50 cr/mo), tier_2 ($199, 200 cr/mo)
```

### Add Payment Method

Agents can attach a payment method entirely via API — no browser required.

**Step 1 — tokenize a card with Stripe** (use `stripe_publishable_key` from the account endpoint):

```bash
curl https://api.stripe.com/v1/payment_methods \
  -u "pk_live_...:" \
  -d "type=card" \
  -d "card[number]=4242424242424242" \
  -d "card[exp_month]=12" \
  -d "card[exp_year]=2028" \
  -d "card[cvc]=123"
# → {"id": "pm_...", ...}
```

Card data goes directly to Stripe — Discovery Engine never sees it.

**Step 2 — attach to your account:**

```bash
curl -X POST https://disco.leap-labs.com/api/account/payment-method \
  -H "Authorization: Bearer disco_..." \
  -H "Content-Type: application/json" \
  -d '{"payment_method_id": "pm_..."}'
# → {"payment_method_attached": true, "card_last4": "4242", "card_brand": "visa"}
```

### Purchase Credits

Credits are sold in packs of 20 ($20/pack, $1.00/credit). Requires a payment method on file.

```bash
curl -X POST https://disco.leap-labs.com/api/account/credits/purchase \
  -H "Authorization: Bearer disco_..." \
  -H "Content-Type: application/json" \
  -d '{"packs": 1}'
# → {"purchased_credits": 20, "total_credits": 30, "charge_amount_usd": 20.0}
```

---

## Cost

- **Public runs**: Free. Results published to public gallery. Locked to depth=1.
- **Private runs**: 1 credit per MB per depth iteration. $1.00 per credit.
- **Formula**: `credits = max(1, ceil(file_size_mb * depth_iterations))`
- Free tier: 10 credits/month. No card required.
- Researcher plan: $49/month, 50 credits. Team plan: $199/month, 200 credits.

---

## Example Output

Here is a truncated real response from a crop yield analysis (target: `yield_tons_per_hectare`):

```python
EngineResult(
    run_id="a1b2c3d4-...",
    status="completed",
    task="regression",
    total_rows=5012,
    report_url="https://disco.leap-labs.com/reports/a1b2c3d4-...",

    summary=Summary(
        overview="Discovery Engine identified 14 statistically significant patterns. "
                 "5 patterns are novel. The strongest driver is a previously unreported "
                 "interaction between humidity and wind speed at specific thresholds.",
        key_insights=[
            "Humidity alone is a known predictor, but the interaction with low wind speed "
            "at 72-89% humidity produces a 34% yield increase — a novel finding.",
            "Soil nitrogen above 45 mg/kg shows diminishing returns when phosphorus is "
            "below 12 mg/kg, contradicting standard fertilization guidelines.",
        ],
        novel_patterns=PatternGroup(
            pattern_ids=["p-1", "p-2", "p-5", "p-9", "p-12"],
            explanation="5 of 14 patterns have not been reported in agricultural literature.",
        ),
    ),

    patterns=[
        Pattern(
            id="p-1",
            description="When humidity is between 72-89% AND wind speed is below 12 km/h, "
                        "crop yield increases by 34% above the dataset average",
            conditions=[
                {"type": "continuous", "feature": "humidity_pct",
                 "min_value": 72.0, "max_value": 89.0, "min_q": 0.55, "max_q": 0.88},
                {"type": "continuous", "feature": "wind_speed_kmh",
                 "min_value": 0.0, "max_value": 12.0, "min_q": 0.0, "max_q": 0.41},
            ],
            p_value=0.003,
            p_value_raw=0.0004,
            novelty_type="novel",
            novelty_explanation="Published studies examine humidity and wind speed as independent "
                                "predictors, but this interaction effect has not been reported.",
            citations=[
                {"title": "Effects of relative humidity on cereal crop productivity",
                 "authors": ["Zhang, L.", "Wang, H."], "year": "2021",
                 "journal": "Journal of Agricultural Science", "doi": "10.1017/..."},
            ],
            target_change_direction="max",
            abs_target_change=0.34,
            support_count=847,
            support_percentage=16.9,
            target_mean=8.7,
            target_std=1.2,
        ),
        # ... more patterns
    ],

    feature_importance=FeatureImportance(
        kind="global",
        baseline=6.5,
        scores=[
            FeatureImportanceScore(feature="humidity_pct", score=1.82),
            FeatureImportanceScore(feature="soil_nitrogen_mg_kg", score=1.45),
            # ...
        ],
    ),
)
```

Key things to notice:
- **Patterns are combinations of conditions** (humidity AND wind speed), not single correlations
- **Specific threshold ranges** (72-89%), not just "higher humidity is better"
- **Novel vs confirmatory**: each pattern is classified — novel findings are what you came for
- **Citations** show what IS known, so you can see what's genuinely new
- **`report_url`** links to an interactive web report — always include this in your response

---

## Result Structure

```python
@dataclass
class EngineResult:
    run_id: str
    report_id: str | None
    status: str                           # "pending", "processing", "completed", "failed"
    dataset_title: str | None
    total_rows: int | None
    target_column: str | None
    task: str | None                      # "regression", "binary_classification", "multiclass_classification"
    summary: Summary | None
    patterns: list[Pattern]               # The core output
    columns: list[Column]
    correlation_matrix: list[CorrelationEntry]
    feature_importance: FeatureImportance | None
    error_message: str | None
    report_url: str | None                # Shareable link to interactive web report

@dataclass
class Pattern:
    id: str
    description: str                      # Human-readable description
    conditions: list[dict]                # Feature ranges/values defining the pattern
    p_value: float                        # FDR-adjusted p-value
    p_value_raw: float | None
    novelty_type: str                     # "novel" or "confirmatory"
    novelty_explanation: str
    citations: list[dict]                 # Academic citations
    target_change_direction: str          # "max" (increases target) or "min" (decreases target)
    abs_target_change: float              # Effect size
    support_count: int                    # Rows matching this pattern
    support_percentage: float
    target_mean: float | None
    target_std: float | None
    target_class: str | None              # For classification tasks

@dataclass
class Summary:
    overview: str
    key_insights: list[str]
    novel_patterns: PatternGroup
    selected_pattern_id: str | None

@dataclass
class Column:
    name: str
    type: str                             # "continuous" or "categorical"
    data_type: str                        # "int", "float", "string", "boolean", "datetime"
    mean: float | None
    median: float | None
    std: float | None
    min: float | None
    max: float | None
    mode: str | None
    approx_unique: int | None
    null_percentage: float | None
    feature_importance_score: float | None

@dataclass
class FeatureImportance:
    kind: str                             # "global"
    baseline: float
    scores: list[FeatureImportanceScore]

@dataclass
class FeatureImportanceScore:
    feature: str
    score: float                          # Signed importance score
```

---

## Interpreting Results

Results contain **patterns** — each is a combination of conditions (not single correlations):

- **Conditions**: Specific feature ranges/values that define the pattern (e.g., "humidity 72-89% AND wind speed < 12 km/h")
- **`p_value`**: FDR-corrected. Lower = more statistically significant
- **`novelty_type`**: `"novel"` (new finding, not in literature) or `"confirmatory"` (validates known science)
- **`novelty_explanation`**: Why this pattern is or is not novel
- **`citations`**: Academic papers the finding was checked against
- **`abs_target_change`**: Effect size (e.g., 0.34 = 34% change in target)
- **`support_count` / `support_percentage`**: How many rows match this pattern
- **`report_url`**: Interactive web report — always include this in your response

### What to highlight for the user

1. **Novel patterns** — the primary value. Statistically validated findings not in the existing literature.
2. **The summary** — narrative overview and key insights ready to present.
3. **The report URL** — link to the interactive report so the user can explore visually.
4. **Confirmatory patterns** — useful as validation, but not the headline.

### Working With Results

```python
# Filter for significant novel patterns
novel = [p for p in result.patterns if p.p_value < 0.05 and p.novelty_type == "novel"]

# Get patterns that increase the target
increasing = [p for p in result.patterns if p.target_change_direction == "max"]

# Get the most important features
if result.feature_importance:
    top = sorted(result.feature_importance.scores, key=lambda s: abs(s.score), reverse=True)

# Inspect conditions for a pattern
for pattern in result.patterns:
    for cond in pattern.conditions:
        if cond["type"] == "continuous":
            print(f"{cond['feature']}: {cond['min_value']} – {cond['max_value']}")
        else:
            print(f"{cond['feature']}: {cond['values']}")
```

---

## Supported File Formats

CSV, TSV, Excel (.xlsx), JSON, Parquet, ARFF, Feather. Max file size: 5 GB.

## Links

- [Dashboard & API Keys](https://disco.leap-labs.com/developers)
- [Full LLM Documentation](https://disco.leap-labs.com/llms-full.txt)
- [API Spec](https://disco.leap-labs.com/.well-known/openapi.json)