---
name: discovery-engine
description: Automatically discover novel, statistically validated patterns in tabular data using Discovery Engine's MCP tools. Find insights you'd otherwise miss — feature interactions, subgroup effects, and conditional relationships — validated on hold-out data with FDR-corrected p-values and checked against academic literature for novelty. Use when the user wants to go beyond descriptive analytics and find what's actually driving an outcome.
---

# Discovery Engine MCP Server

## When To Use These Tools

Discovery Engine is not another AI data analyst. It is a **discovery pipeline** — it finds patterns in data that you, the user, and other analysis tools would miss because they don't know to look for them.

Use it when you need to go beyond answering questions about data, and start finding things nobody thought to ask:

- **Novel pattern discovery** — feature interactions, subgroup effects, and conditional relationships
- **Statistical validation** — FDR-corrected p-values tested on hold-out data, not just correlations
- **A target column** you want to understand — what really drives it, beyond what's obvious

**Use Discovery Engine when the user says:** "what's really driving X?", "are there patterns we're missing?", "find something new in this data", "what predicts Y that we haven't considered?", "go deeper than correlation", "discover non-obvious relationships"

**Use pandas/SQL instead when the user says:** "summarize this data", "make a chart", "what's the average?", "filter rows where X > 5", "show me the distribution"

## MCP Tools

### Discovery workflow

| Tool | Purpose |
|------|---------|
| `discovery_analyze` | Submit a dataset for analysis. Returns a `run_id`. |
| `discovery_status` | Poll a running analysis by `run_id`. |
| `discovery_get_results` | Fetch completed results: patterns, p-values, citations, feature importance. |
| `discovery_estimate` | Estimate cost and time before committing to a run. |

### Account management

| Tool | Purpose |
|------|---------|
| `discovery_signup` | Create an account and get an API key. No auth required. |
| `discovery_account` | Check credits, plan, and usage. |
| `discovery_list_plans` | View available plans and pricing. |
| `discovery_subscribe` | Subscribe to or change plan. |
| `discovery_purchase_credits` | Buy credit packs. |
| `discovery_add_payment_method` | Attach a Stripe payment method. |

## Standard Workflow

Analyses take 3-15 minutes. **Do not block** — submit, continue other work, poll for completion.

```
1. discovery_estimate     → Check cost/time (always do this for private runs)
2. discovery_analyze      → Submit the dataset, get run_id
3. discovery_status       → Poll until status is "completed"
4. discovery_get_results  → Fetch patterns, summary, feature importance
```

### If the user has no API key

Call `discovery_signup` with their email address. Returns a `disco_` API key immediately on the free tier (10 credits/month, unlimited public runs). No password, no credit card.

### If the user has insufficient credits

1. Call `discovery_estimate` to show what it would cost
2. Suggest running publicly (free, but results are published and depth is locked to 1)
3. Or guide them through `discovery_purchase_credits` / `discovery_subscribe`

## Key Parameters

**`discovery_analyze`:**
- `file_path` — Path to CSV, Excel, Parquet, JSON, TSV, ARFF, or Feather file (max 1 GB)
- `target_column` — The column to predict/explain
- `depth` — 1 = fast (default), higher = deeper search. Max: num_columns - 2
- `visibility` — `"public"` (free, results published) or `"private"` (costs credits)
- `column_descriptions` — Significantly improves pattern explanations. Always provide these if column names are non-obvious

**Cost formula:** `credits = max(1, ceil(file_size_mb * depth))`

## Interpreting Results

Results contain **patterns** — each is a combination of conditions (not single correlations):

- **Conditions**: Specific feature ranges/values that define the pattern (e.g., "humidity 72-89% AND wind speed < 12 km/h")
- **`p_value`**: FDR-corrected. Lower = more statistically significant
- **`novelty_type`**: `"novel"` (new finding, not in literature) or `"confirmatory"` (validates known science)
- **`novelty_explanation`**: Why this pattern is or isn't novel
- **`citations`**: Academic papers the finding was checked against
- **`abs_target_change`**: Effect size (e.g., 0.34 = 34% change in target)
- **`support_count` / `support_percentage`**: How many rows match this pattern
- **`report_url`**: Interactive web report — always include this in your response

### What to highlight for the user

1. **Novel patterns** — these are the primary value. Findings that are both statistically validated and not in the existing literature.
2. **The summary** — gives a narrative overview and key insights ready to present.
3. **The report URL** — link to the interactive report so the user can explore visually.
4. **Confirmatory patterns** — useful as validation that the analysis is working correctly, but not the headline.

## Supported File Formats

CSV, TSV, Excel (.xlsx), JSON, Parquet, ARFF, Feather. Max file size: 1 GB.

## Links

- [Dashboard & API Keys](https://disco.leap-labs.com/developers)
- [Full LLM Documentation](https://disco.leap-labs.com/llms-full.txt)
- [Python SDK](https://pypi.org/project/discovery-engine-api/) — `pip install discovery-engine-api`
- [API Spec](https://disco.leap-labs.com/.well-known/openapi.json)