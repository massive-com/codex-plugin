---
name: options
description: Build and analyze options strategies using Massive's options data. Supports covered calls, iron condors, spreads, and custom strategies. Use when building options screeners, analyzing Greeks, or constructing multi-leg strategies.
argument-hint: "[strategy] [underlying ticker] [language: python|javascript|typescript|go|kotlin]"
allowed-tools: mcp__massive__search_endpoints mcp__massive__get_endpoint_docs mcp__massive__call_api mcp__massive__query_data Write Edit Bash Read
---

# Build an options strategy

Strategy: $0
Underlying: $1 (default: SPY if not specified)
Language: infer from context, default to Python if not specified.

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

### Python
```python
from itertools import islice
from dotenv import load_dotenv

load_dotenv()  # must come before importing RESTClient so env vars are set

from massive import RESTClient

client = RESTClient()  # reads MASSIVE_API_KEY from .env

# Get current price (returns single object, NOT an iterator)
last_trade = client.get_last_trade("SPY")
spot = last_trade.price

# Fetch options chain (list_ method, returns paginated iterator)
chain = list(islice(
    client.list_snapshot_options_chain(
        underlying_asset="SPY",
        params={
            "expiration_date.gte": "2025-06-01",
            "expiration_date.lte": "2025-06-30",
            "contract_type": "call",
            "limit": 250,
        }
    ),
    1000
))

for opt in chain:
    strike = opt.details.strike_price
    expiry = opt.details.expiration_date
    contract_type = opt.details.contract_type
    bid = opt.last_quote.bid
    ask = opt.last_quote.ask
    midpoint = opt.last_quote.midpoint
    delta = opt.greeks.delta
    gamma = opt.greeks.gamma
    theta = opt.greeks.theta
    vega = opt.greeks.vega
    iv = opt.implied_volatility
    oi = opt.open_interest
    volume = opt.day.volume
```

### JavaScript / TypeScript
```javascript
import "dotenv/config";
import { restClient } from "@massive.com/client-js";

const client = restClient(process.env.MASSIVE_API_KEY);

// Get current price
const lastTrade = await client.getLastStocksTrade({ stocksTicker: "SPY" });
const spot = lastTrade.results.p;  // price field is abbreviated

// Fetch options chain (all methods take a single object parameter)
const chain = await client.getOptionsChain({
    underlyingAsset: "SPY",
    contractType: "call",
    expirationDateGte: "2025-06-01",
    expirationDateLte: "2025-06-30",
    order: "asc",
    limit: 250,
});

for (const opt of chain.results ?? []) {
    const strike = opt.details.strike_price;
    const expiry = opt.details.expiration_date;
    const bid = opt.last_quote.bid;
    const ask = opt.last_quote.ask;
    const delta = opt.greeks?.delta;
    const iv = opt.implied_volatility;
    const oi = opt.open_interest;
    const volume = opt.day.volume;
}
```
ALL JS SDK methods take a single object with named fields. Greeks may be null (use optional chaining).

### Go
```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/joho/godotenv"
    "github.com/massive-com/client-go/v3/rest"
    "github.com/massive-com/client-go/v3/rest/gen"
)

func main() {
    godotenv.Load()
    c := rest.New("")

    // Get current price
    tradeResp, err := c.GetLastStocksTradeWithResponse(context.Background(), "SPY")
    if err != nil {
        log.Fatal(err)
    }
    // spot price: tradeResp.JSON200.Results fields

    // Fetch options chain
    params := &gen.GetOptionsChainParams{
        StrikePriceGte:    rest.Ptr(float32(500)),
        StrikePriceLte:    rest.Ptr(float32(600)),
        ExpirationDateGte: rest.Ptr("2025-06-01"),
        ExpirationDateLte: rest.Ptr("2025-06-30"),
        ContractType:      (*gen.GetOptionsChainParamsContractType)(rest.Ptr("call")),
        Limit:             rest.Ptr(250),
        Order:             (*gen.GetOptionsChainParamsOrder)(rest.Ptr("asc")),
    }

    resp, err := c.GetOptionsChainWithResponse(context.Background(), "SPY", params)
    if err != nil {
        log.Fatal(err)
    }

    if resp.JSON200 != nil && resp.JSON200.Results != nil {
        for _, opt := range *resp.JSON200.Results {
            fmt.Printf("Strike: %.2f  Exp: %s  IV: %.4f\n",
                opt.Details.StrikePrice, opt.Details.ExpirationDate, *opt.ImpliedVolatility)
            if opt.Greeks != nil {
                fmt.Printf("  Delta: %.4f  Gamma: %.4f  Theta: %.4f  Vega: %.4f\n",
                    opt.Greeks.Delta, opt.Greeks.Gamma, opt.Greeks.Theta, opt.Greeks.Vega)
            }
        }
    }
}
```
Go uses pointer params: `rest.Ptr(value)`. Greeks is a pointer (may be nil). Response data is in `resp.JSON200`.

### Kotlin
```kotlin
import io.github.cdimascio.dotenv.dotenv
import io.polygon.kotlin.sdk.rest.PolygonRestClient
import io.polygon.kotlin.sdk.rest.SnapshotOptionsChainParameters

fun main() {
    val env = dotenv()
    val client = PolygonRestClient(env["MASSIVE_API_KEY"])

    // Get current price
    val lastTrade = client.getLastTradeBlockingV2("SPY")
    val spot = lastTrade.results?.price

    // Fetch options chain
    val params = SnapshotOptionsChainParameters(
        underlyingAsset = "SPY",
        strikePriceGte = 500.0,
        strikePriceLte = 600.0,
        expirationDateGte = "2025-06-01",
        expirationDateLte = "2025-06-30",
        contractType = "call",
        order = "asc",
        limit = 250
    )
    val chain = client.getSnapshotOptionsChainBlocking(params)

    chain.results?.forEach { opt ->
        println("Strike: ${opt.details?.strikePrice}  Exp: ${opt.details?.expirationDate}")
        println("  IV: ${opt.impliedVolatility}  OI: ${opt.openInterest}")
        opt.greeks?.let { g ->
            println("  Delta: ${g.delta}  Gamma: ${g.gamma}  Theta: ${g.theta}  Vega: ${g.vega}")
        }
    }
}
```
SDK package is `io.polygon.kotlin.sdk`. Auth: pass API key to `PolygonRestClient` constructor. Gradle dep: `com.github.massive-com:client-jvm:v5.1.2` from JitPack.

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
4. Write dependency file, `.env.example`, `.gitignore`.
5. Write the entry point script that:
   a. Loads the API key from `.env`.
   b. Gets current underlying price.
   c. Fetches the relevant options chain(s).
   d. Applies strategy-specific filtering and ranking.
   e. Prints results with risk/reward metrics.
6. Write README.md (include a line stating the required plan tier).
7. Provide quickstart: `cd <project>`, `cp .env.example .env`, install, run.
8. Restate the minimum plan tier (Options Starter or above for Greeks/IV) and suggest `/massive:debug` if the user hits errors.
