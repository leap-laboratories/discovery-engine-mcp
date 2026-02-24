# Discovery Engine MCP Server

MCP server for [Discovery Engine](https://www.leap-labs.com) by Leap Laboratories. Superhuman exploratory data analysis – finds insight you wouldn't think to look for. Novel, statistically validated patterns surfaced automatically — feature interactions, subgroup effects, and conditional relationships you'd otherwise miss. Multiple scientific discoveries to date: free for open data. 

## Quick Start

```bash
pip install discovery-engine-mcp
```

Set your API key and run:

```bash
export DISCOVERY_API_KEY=disco_...
discovery-engine-mcp
```

Or run with `uvx` (no install needed):

```bash
DISCOVERY_API_KEY=disco_... uvx discovery-engine-mcp
```

### Claude Desktop / Cursor

Add to your MCP config:

```json
{
  "mcpServers": {
    "discovery-engine": {
      "command": "uvx",
      "args": ["discovery-engine-mcp"],
      "env": {
        "DISCOVERY_API_KEY": "disco_..."
      }
    }
  }
}
```

### Remote (Streamable HTTP)

Connect directly to the hosted server — no local install needed:

```
URL: https://disco.leap-labs.com/mcp
Auth: Bearer disco_...
```

## Getting an API Key

Sign up from the command line (no password, no credit card):

```bash
curl -X POST https://disco.leap-labs.com/v1/signup \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com"}'
```

Or create a key at [disco.leap-labs.com/developers](https://disco.leap-labs.com/developers).

Free tier: 10 credits/month for private runs, unlimited public runs.

## Tools

### Discovery

| Tool | Description |
|------|-------------|
| `discovery_analyze` | Run Discovery Engine on tabular data to find novel patterns |
| `discovery_status` | Check the status of a running analysis |
| `discovery_get_results` | Fetch full results (patterns, p-values, citations, feature importance) |
| `discovery_estimate` | Estimate cost and time before running |

### Account management

| Tool | Description |
|------|-------------|
| `discovery_signup` | Create an account and get an API key (no auth required) |
| `discovery_account` | Check account status, credits, and plan |
| `discovery_list_plans` | List available plans with pricing |
| `discovery_subscribe` | Subscribe to or change your plan |
| `discovery_purchase_credits` | Buy credit packs |
| `discovery_add_payment_method` | Attach a Stripe payment method |

## Configuration

Set `DISCOVERY_API_KEY` to your API key (starts with `disco_`).

## Links

- [Full Documentation (LLM-friendly)](https://disco.leap-labs.com/llms-full.txt)
- [Python SDK](https://pypi.org/project/discovery-engine-api/) — `pip install discovery-engine-api`
- [Agent Integration Guide](https://disco.leap-labs.com/agents)
- [API Keys & Dashboard](https://disco.leap-labs.com/developers)
- [OpenAPI Spec](https://disco.leap-labs.com/.well-known/openapi.json)
- [MCP Manifest](https://disco.leap-labs.com/.well-known/mcp.json)

## License

MIT
