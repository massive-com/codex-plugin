---
name: massive-options
description: Build and analyze options strategies using Massive's options data. Use when building Massive-backed options screeners, Greek-aware analysis, or multi-leg strategies. Do not use for generic investing advice or non-Massive tooling.
---

# Build an options strategy

Strategy: `$0`
Underlying: `$1` (default: SPY if not specified)
Language: infer from context, default to Python if not specified.

Prefer official Massive SDK methods and live expiration windows. Do not hardcode expired contracts or invent chain fields that are not documented in the SDK patterns below.

**Keep the generated code minimal.** Write what the strategy needs; do not add env-var configuration tables, helper utility modules, or abstractions (`require_env`, `choose_option`, type-dispatch helpers) the user didn't ask for. A working 40-line script is better than a 150-line scaffold with framework-like hooks.

## Plan tier check (run BEFORE scaffolding)

Options strategies rely on Greeks (delta, gamma, theta, vega) and implied volatility. These fields are **only populated on Options Starter ($49/mo) or above**. On the Basic Options plan, the options chain returns without Greeks/IV, and any strategy that filters or ranks on those fields will return empty results.

Before scaffolding:

- Confirm the user has **Options Starter or above**. If they have not said, ask once: "Options strategies need Greeks and IV, which require Options Starter ($49/mo) or above. Are you on Options Starter or higher?"
- If they say Basic: offer a downgraded build that uses only open interest, volume, bid/ask, and strike/expiration filtering (no Greek-based filtering). Note this limitation in the generated README.
- If they say Starter or higher: proceed normally.

## Output: a runnable project

Create a project directory with the strategy name (e.g., `spy-bull-call-spread/`). The project must include:

1. **Dependency file** (`pyproject.toml`, `package.json`, `go.mod`, `build.gradle.kts`). The Python package is `massive` on PyPI (NOT `massive-sdk` or `massive-api`). Pin `massive>=2.4.0`.
2. **`.env.example`** with `MASSIVE_API_KEY=your_api_key_here`
3. **`.gitignore`** with `.env` and language-specific entries
4. **Entry point** that implements the strategy
5. **README.md** with quickstart instructions

End with:

```
cd <project-name>
cp .env.example .env
# Add your Massive API key to .env
```

Then the language-specific install and run commands.

## Options ticker format

OCC symbology: `O:AAPL250117C00150000`

- `O:` prefix
- `AAPL` underlying (padded to 6 chars in OCC, but SDK handles this)
- `250117` expiration (YYMMDD)
- `C` or `P` for call/put
- `00150000` strike price * 1000 (8 digits)

## Expiration-date guidance

The snapshot options chain only returns **live, unexpired contracts**. When generating code, derive expiration dates from the current date, not from the template literal dates below. A reasonable default window is `today + 7d` to `today + 60d`:

```python
from datetime import date, timedelta
today = date.today()
expiry_lo = (today + timedelta(days=7)).isoformat()
expiry_hi = (today + timedelta(days=60)).isoformat()
```

If the user specifies an expiration window, honor it, but warn if the window is already in the past.

## SDK patterns for options chain

After resolving the language, read the matching reference for a working options-chain fetch pattern (last trade + chain params + field access):

- **Python** → [references/python.md](references/python.md)
- **JavaScript / TypeScript** → [references/javascript.md](references/javascript.md)
- **Go** → [references/go.md](references/go.md)
- **Kotlin / JVM** → [references/kotlin.md](references/kotlin.md)

Only read the reference for the chosen language.

## Strategy templates

### Covered call

1. Fetch current stock price via last trade
2. Fetch call options chain for target expiration
3. Filter: OTM calls (strike > current price), delta between 0.15-0.40
4. Rank by: premium/strike ratio, annualized return, probability OTM

### Iron condor

1. Fetch both call and put chains for target expiration
2. Short call: delta ~0.15-0.20 (OTM)
3. Long call: 1-2 strikes above short call (wing protection)
4. Short put: delta ~(-0.15 to -0.20) (OTM)
5. Long put: 1-2 strikes below short put (wing protection)
6. Calculate: max profit (net credit), max loss (wing width - credit), breakevens

### Vertical spread (bull call / bear put)

1. Fetch chain for target expiration and contract type
2. Buy lower strike, sell higher strike (bull call) or vice versa
3. Calculate: max profit, max loss, breakeven, risk/reward ratio

### Custom screening

1. Fetch full chain
2. Apply user-specified filters (delta range, IV threshold, OI minimum, bid-ask spread)
3. Sort and rank by user criteria
4. Output as formatted table or DataFrame

## MCP server tools

For exploring data before writing code, the MCP tools can help:

1. `call_api` with `/v3/snapshot/options/{underlying}` and `store_as: "chain"`
2. `query_data` with SQL to filter and rank:

   ```sql
   SELECT * FROM chain
   WHERE greeks_delta BETWEEN 0.15 AND 0.40
   AND implied_volatility > 0.3
   ORDER BY day_volume DESC
   LIMIT 20
   ```

Built-in financial functions available via `apply` parameter: Black-Scholes (`bs_price`, `bs_delta`, `bs_gamma`, `bs_theta`, `bs_vega`, `bs_rho`), returns (`simple_return`, `log_return`, `cumulative_return`, `sharpe_ratio`, `sortino_ratio`).

## Steps

1. **Plan check first** (see "Plan tier check" above). Confirm Options Starter or above before using Greek-based filters.
2. Determine which strategy the user wants from `$0`.
3. Create the project directory.
4. Resolve the language and read the matching reference for the options-chain SDK pattern.
5. Write dependency file, `.env.example`, `.gitignore`.
6. Write the entry point script that:
   a. Loads the API key from `.env`.
   b. Gets current underlying price.
   c. Fetches the relevant options chain(s).
   d. Applies strategy-specific filtering and ranking.
   e. Prints results with risk/reward metrics.
7. Write README.md (include a line stating the required plan tier).
8. Provide quickstart: `cd <project>`, `cp .env.example .env`, install, run.
9. Restate the minimum plan tier (Options Starter or above for Greeks/IV) and suggest `$massive-debug` if the user hits errors.
