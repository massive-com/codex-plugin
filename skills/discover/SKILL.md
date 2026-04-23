---
name: discover
description: Find the right Massive API endpoint for a financial data task. Use when the user needs to find which endpoint returns specific market data, or when exploring what data is available for a given asset class or use case.
argument-hint: "[description of data needed]"
allowed-tools: mcp__massive__search_endpoints mcp__massive__get_endpoint_docs Read
---

# Find the right Massive endpoint

The user needs: $ARGUMENTS

## Process

1. Use `search_endpoints` to find candidate endpoints matching the user's description. Try multiple search terms if the first query returns few results. For example, if "options Greeks" returns little, also try "options chain snapshot."

2. For the top 2-3 matches, use `get_endpoint_docs` to pull full parameter documentation.

3. **Detect the user's language** from context (open files, project type, or explicit mention). Default to Python if unclear.

4. Present results to the user in this format for each relevant endpoint:

   **Endpoint:** `GET /v2/aggs/ticker/{ticker}/range/{multiplier}/{timespan}/{from}/{to}`
   **What it returns:** OHLCV candlestick bars for any supported ticker.
   **Key parameters:** `ticker`, `multiplier`, `timespan` (second/minute/hour/day/week/month/quarter/year), `from`/`to` (YYYY-MM-DD or ms epoch), `adjusted` (default true), `sort` (asc/desc).
   **Plan tier:** Available on all plans.

   Then show the SDK example in the user's language.

5. If the data need spans multiple endpoints, explain how they fit together.

6. Note plan tier requirements. Key thresholds:
   - Basic (free): end-of-day data, aggs, reference, technicals, corporate actions. 5 calls/min.
   - Starter ($29-49/mo): adds WebSockets, flat files, snapshots, second aggs. Options Starter adds Greeks/IV.
   - Developer ($79/mo): adds trades.
   - Advanced ($99-199/mo): adds quotes, real-time data. Stocks Advanced adds financials/ratios.
   - Business ($999-2,500/mo): commercial use, FMV, no exchange fees.
   - Partner data (Benzinga, ETF Global, TMX): separate add-ons, $99/mo per dataset.

## SDK examples by language

Show the example matching the user's language. Always include .env loading.

### Python
```python
from itertools import islice
from dotenv import load_dotenv

load_dotenv()  # must come before importing RESTClient so env vars are set

from massive import RESTClient

client = RESTClient()  # reads MASSIVE_API_KEY from .env
bars = list(islice(client.list_aggs("AAPL", 1, "day", "2025-01-01", "2025-06-01", sort="asc"), 200))
```
Method naming: `list_aggs`, `list_universal_snapshots(ticker_any_of=[...])`, `list_snapshot_options_chain`, `get_last_trade`, `get_sma`/`get_rsi` (return `SingleIndicatorResults`, access `.values`). Pagination: iterator-based, use `islice()` to cap.

### JavaScript / TypeScript
```javascript
import "dotenv/config";
import { restClient } from "@massive.com/client-js";

const client = restClient(process.env.MASSIVE_API_KEY);
const response = await client.getStocksAggregates({
  stocksTicker: "AAPL", multiplier: 1, timespan: "day",
  from: "2025-01-01", to: "2025-06-01", adjusted: true, sort: "asc", limit: 200,
});
for (const bar of response.results ?? []) {
  console.log(bar.o, bar.h, bar.l, bar.c, bar.v, new Date(bar.t));
}
```
ALL methods take a single object parameter with named fields. Method naming: `getStocksAggregates`, `getOptionsChain`, `getLastStocksTrade({ stocksTicker })`, `getSnapshots({ tickerAnyOf: "AAPL,X:BTCUSD" })`. Bar fields abbreviated (`o`, `h`, `l`, `c`, `v`, `t`). Pagination: `{ pagination: true }` as third arg to `restClient()`.

### Go
```go
import (
    "github.com/joho/godotenv"
    "github.com/massive-com/client-go/v3/rest"
    "github.com/massive-com/client-go/v3/rest/gen"
)

godotenv.Load()
c := rest.New("") // reads MASSIVE_API_KEY from env

params := &gen.GetStocksAggregatesParams{Sort: "asc", Limit: rest.Ptr(200)}
resp, err := c.GetStocksAggregatesWithResponse(ctx, "AAPL", 1, gen.Day, "2025-01-01", "2025-06-01", params)
for _, bar := range *resp.JSON200.Results {
    fmt.Println(bar.O, bar.H, bar.L, bar.C, bar.V, bar.Timestamp)
}
```
Method naming: `GetStocksAggregatesWithResponse`, `GetOptionsChainWithResponse`, `GetLastStocksTradeWithResponse`, `GetSnapshotsWithResponse`. Response data in `resp.JSON200`. Pointer helpers: `rest.Ptr(value)`.

### Kotlin
```kotlin
import io.github.cdimascio.dotenv.dotenv
import io.polygon.kotlin.sdk.rest.PolygonRestClient
import io.polygon.kotlin.sdk.rest.AggregatesParameters

val env = dotenv()
val client = PolygonRestClient(env["MASSIVE_API_KEY"])

val result = client.getAggregatesBlocking(AggregatesParameters(
    ticker = "AAPL", multiplier = 1, timespan = "day",
    fromDate = "2025-01-01", toDate = "2025-06-01", sort = "asc", limit = 200
))
result.results?.forEach { bar -> println("${bar.open} ${bar.high} ${bar.low} ${bar.close} ${bar.volume}") }
```
SDK package: `io.polygon.kotlin.sdk`. Pass API key to `PolygonRestClient` constructor. Bar fields use full names (`open`, `high`, `low`, `close`, `volume`, `timestampMillis`). Gradle dep: `com.github.massive-com:client-jvm:v5.1.2` from JitPack.

## Ticker prefix reminder

- Equities: plain (`AAPL`)
- Crypto: `X:` prefix (`X:BTCUSD`)
- Forex: `C:` prefix (`C:EURUSD`)
- Indices: `I:` prefix (`I:SPX`)
- Options: `O:` prefix (`O:AAPL250117C00150000`)
- Futures: product codes (`ES`, `NQ`)
