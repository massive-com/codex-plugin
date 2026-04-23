# Massive API

Massive is a financial data API platform providing accurate, real-time pricing data across equities, options, crypto, forex, indices, and futures. Available via REST, WebSocket, and S3 flat files.

**Official SDKs:**
- Python: `massive` on PyPI (v2.5.0, Python 3.9+) / https://github.com/massive-com/client-python
- JavaScript/TypeScript: `@massive.com/client-js` on npm (v10.6.0+, Node.js 16+) / https://github.com/massive-com/client-js
- Go: `github.com/massive-com/client-go/v3` (Go 1.21+) / https://github.com/massive-com/client-go
- Kotlin/JVM: `client-jvm` v5.1.2 via JitPack (JDK 21+, Android SDK 21+) / https://github.com/massive-com/client-jvm

**Base URL:** `https://api.massive.com`
**Auth:** API key passed via `Authorization: Bearer <key>` header, or via SDK constructor/environment variable.

## Reliability protocol

- Prefer the bundled Massive MCP tools and current Massive docs for endpoint names, parameters, and other details that can drift over time.
- Use this `AGENTS.md` for stable guidance: ticker formats, SDK naming, auth patterns, plan tiers, pagination behavior, and common pitfalls.
- If live endpoint lookup is unavailable, say that current verification is unavailable and avoid inventing routes, parameters, plan entitlements, or SDK behavior.
- Prefer official Massive SDK methods over raw HTTP whenever the SDK already covers the workflow.

## Ticker conventions

- Equities: plain symbols (`AAPL`, `MSFT`, `NVDA`, `TSLA`)
- Crypto: `X:` prefix (`X:BTCUSD`, `X:ETHUSD`, `X:SOLUSD`)
- Forex: `C:` prefix (`C:EURUSD`, `C:GBPUSD`, `C:USDJPY`)
- Indices: `I:` prefix (`I:SPX`, `I:NDX`, `I:DJI`, `I:VIX`)
- Options: `O:` prefix with OCC symbology (`O:AAPL250117C00150000`)
- Futures: product codes (`ES`, `NQ`, `CL`, `GC`) or contract tickers (`ESM6`)

The universal snapshot endpoint (`/v3/snapshot`) accepts mixed-asset tickers in a single request:
```python
snapshots = client.list_universal_snapshots(ticker_any_of=["AAPL", "X:BTCUSD", "C:EURUSD", "I:SPX"])
```

## Python SDK patterns

```python
from dotenv import load_dotenv

load_dotenv()  # must come before importing RESTClient so env vars are set

from massive import RESTClient, WebSocketClient

# Client reads MASSIVE_API_KEY from env automatically
client = RESTClient()

# Or pass explicitly
client = RESTClient(api_key="your_key")

# Debug options
client = RESTClient(trace=True, verbose=True)  # log request/response details
```

**Pagination is iterator-based.** All `list_` methods return generators that auto-paginate. The `limit` parameter controls page size, not total results. Use `islice()` to cap:

```python
from itertools import islice

# Get 100 days of daily bars
bars = list(islice(client.list_aggs("AAPL", 1, "day", "2025-01-01", "2025-06-01"), 100))
for bar in bars:
    print(bar.open, bar.high, bar.low, bar.close, bar.volume, bar.timestamp)
```

To disable auto-pagination: `RESTClient(api_key=key, pagination=False)`.

**`get_*` methods return single objects, not iterators.** Do NOT wrap them in `list()` or `islice()`. This includes `get_last_trade()`, `get_last_quote()`, `get_market_status()`, `get_market_holidays()`, `get_real_time_currency_conversion()`, and the technical indicator methods (`get_sma`, `get_ema`, `get_rsi`, `get_macd`).

**Timestamps:** Aggregates return millisecond epoch in `bar.timestamp`. Trades use nanosecond epoch in `trade.sip_timestamp`. Convert with:
```python
import pandas as pd
dt = pd.to_datetime(bar.timestamp, unit="ms", utc=True)
```

**Key SDK methods (verified from v2.5.0 source):**
- `list_aggs(ticker, multiplier, timespan, from_, to)` - OHLCV bars (adjusted by default)
- `list_universal_snapshots(ticker_any_of=[...])` - live snapshot across all asset classes
- `list_snapshot_options_chain(underlying_asset, params={})` - options chain with Greeks and IV
- `get_last_trade(ticker)` / `get_last_quote(ticker)` - most recent trade/quote
- `list_trades(ticker)` / `list_quotes(ticker)` - historical trades/quotes
- `get_sma()`, `get_ema()`, `get_rsi()`, `get_macd()` - technical indicators (return `SingleIndicatorResults`, not iterators; access `.values` for the list of data points, each with `.timestamp` and `.value`)
- `list_stocks_splits(ticker)`, `list_stocks_dividends(ticker)` - corporate actions
- `list_financials_balance_sheets()`, `list_financials_income_statements()`, `list_financials_cash_flow_statements()`, `list_financials_ratios()` - fundamentals (Stocks Advanced+)
- `list_short_interest()`, `list_short_volume()`, `list_stocks_floats()` - market microstructure
- `list_treasury_yields()`, `list_inflation()`, `list_labor_market_indicators()` - economy/fed data
- `get_market_status()`, `get_market_holidays()` - market operations
- `get_real_time_currency_conversion()` - forex conversion
- `list_stocks_filings_risk_factors(ticker)` - SEC filing risk factors
- `list_futures_aggregates()`, `list_futures_contracts()`, `list_futures_trades()` - futures
- `list_benzinga_news()`, `list_benzinga_ratings()`, `list_benzinga_earnings()` - partner data (add-on)
- `get_etf_global_constituents()`, `get_etf_global_fund_flows()` - ETF data (add-on)
- `list_tmx_corporate_events()` - corporate events (add-on)

**Filter operators:** Many `list_` methods accept a `params` dict with comparison operators:
```python
# Options chain with filters
chain = client.list_snapshot_options_chain("AAPL", params={
    "expiration_date.gte": "2025-06-01",
    "strike_price.gte": 150,
    "strike_price.lte": 200,
})
```
Supported operators: `.gt`, `.gte`, `.lt`, `.lte`, `.any_of`.

## JavaScript/TypeScript SDK patterns

```javascript
import "dotenv/config";
import { restClient, websocketClient } from "@massive.com/client-js";

const client = restClient(process.env.MASSIVE_API_KEY);

// With auto-pagination (follows next_url automatically)
const client = restClient(process.env.MASSIVE_API_KEY, "https://api.massive.com", { pagination: true });
```

**ALL methods take a single object parameter with named fields.** Do NOT use positional arguments.

**Key method examples:**
```javascript
// Aggregates
await client.getStocksAggregates({ stocksTicker: "AAPL", multiplier: 1, timespan: "day", from: "2025-01-01", to: "2025-06-01", sort: "asc" });
await client.getCryptoAggregates({ cryptoTicker: "X:BTCUSD", multiplier: 1, timespan: "day", from: "...", to: "..." });

// Last trade/quote
await client.getLastStocksTrade({ stocksTicker: "AAPL" });
await client.getLastStocksQuote({ stocksTicker: "AAPL" });

// Universal snapshot (tickerAnyOf is a comma-separated STRING, not an array)
await client.getSnapshots({ tickerAnyOf: "AAPL,X:BTCUSD,C:EURUSD,I:SPX" });

// Options chain
await client.getOptionsChain({ underlyingAsset: "AAPL", contractType: "call", expirationDateGte: "2025-06-01", strikePriceGte: 150, strikePriceLte: 200, limit: 250 });

// Technical indicators (note: stockTicker, not stocksTicker)
await client.getStocksSMA({ stockTicker: "AAPL", timespan: "day", window: 50, seriesType: "close" });
await client.getStocksRSI({ stockTicker: "AAPL", timespan: "day", window: 14, seriesType: "close" });

// Market status (no params needed)
await client.getMarketStatus();
```

**Bar fields are abbreviated:** `o` (open), `h` (high), `l` (low), `c` (close), `v` (volume), `t` (Unix ms timestamp), `vw` (VWAP), `n` (transactions).

**Timestamps:** Aggregates `t` field is milliseconds. Convert with `new Date(bar.t)`. Trades/quotes use nanoseconds; convert with `new Date(trade.t / 1_000_000)`.

**Pagination:** NOT iterator-based. Pass `{ pagination: true }` as third arg to `restClient()` to auto-accumulate all pages. Without it, check `response.next_url` for manual pagination.

**WebSocket:**
```javascript
const ws = websocketClient(process.env.MASSIVE_API_KEY, "wss://delayed.massive.com");
const stocksWS = ws.stocks();
stocksWS.onopen = () => stocksWS.send(JSON.stringify({ action: "subscribe", params: "T.AAPL" }));
stocksWS.onmessage = ({ data }) => {
  for (const msg of JSON.parse(data)) {
    if (msg.ev === "T") console.log(msg.p, msg.s); // price, size
  }
};
```
Returns W3CWebSocket instances. Market connections: `ws.stocks()`, `ws.options()`, `ws.crypto()`, `ws.forex()`, `ws.indices()`, `ws.futures()`. Message events: `T` (trade), `Q` (quote), `A` (per-second agg), `AM` (per-minute agg).

## Go SDK patterns

```go
import (
    "github.com/joho/godotenv"
    "github.com/massive-com/client-go/v3/rest"
    "github.com/massive-com/client-go/v3/rest/gen"
)

godotenv.Load()           // loads .env
c := rest.New("")          // reads MASSIVE_API_KEY from env (panics if empty)
c := rest.New("your_key")  // or pass explicitly
```

**All REST methods follow the pattern `{OperationName}WithResponse`.** Response data is in `resp.JSON200`. Results are pointer to slice (`*[]T`).

**Key method mappings:**
| Python | Go |
|---|---|
| `list_aggs(...)` | `c.GetStocksAggregatesWithResponse(ctx, ticker, mult, gen.Day, from, to, params)` |
| `list_universal_snapshots(...)` | `c.GetSnapshotsWithResponse(ctx, &gen.GetSnapshotsParams{TickerAnyOf: rest.Ptr("AAPL,X:BTCUSD")})` |
| `list_snapshot_options_chain(...)` | `c.GetOptionsChainWithResponse(ctx, underlying, params)` |
| `get_last_trade(ticker)` | `c.GetLastStocksTradeWithResponse(ctx, ticker)` |

**Pointer helpers:** Use `rest.Ptr(value)` for optional params. Example: `rest.Ptr(true)`, `rest.Ptr(200)`, `rest.Ptr("asc")`.

**Timestamps:** Aggregates `Timestamp` field is Unix milliseconds (`int`). Convert with `time.UnixMilli(int64(bar.Timestamp))`. Trades use nanoseconds.

**Pagination:** Auto-pagination is on by default. Disable with `rest.NewWithOptions("", rest.WithPagination(false))`.

**Nil safety:** Always check `resp.JSON200 != nil && resp.JSON200.Results != nil` before dereferencing. Greeks and other optional fields are pointers.

**WebSocket:**
```go
import (
    massivews "github.com/massive-com/client-go/v3/websocket"
    "github.com/massive-com/client-go/v3/websocket/models"
)

c, _ := massivews.New(massivews.Config{
    APIKey: os.Getenv("MASSIVE_API_KEY"),
    Feed:   massivews.Delayed,
    Market: massivews.Stocks,
})
c.Subscribe(massivews.StocksTrades, "AAPL")
c.Connect()
for out := range c.Output() {
    switch t := out.(type) {
    case models.EquityTrade:
        fmt.Println(t.Price, t.Size)
    }
}
```
Channel-based: `c.Output()` for messages, `c.Error()` for fatal errors. Topics: `massivews.StocksTrades`, `massivews.StocksQuotes`, `massivews.StocksMinAggs`, etc.

## Kotlin/JVM SDK patterns

```kotlin
import io.github.cdimascio.dotenv.dotenv
import io.polygon.kotlin.sdk.rest.PolygonRestClient
import io.polygon.kotlin.sdk.rest.AggregatesParameters

val env = dotenv()
val client = PolygonRestClient(env["MASSIVE_API_KEY"])  // API key is a constructor param
```

**Gradle dependency** (JitPack, NOT Maven Central):
```kotlin
repositories {
    maven("https://jitpack.io")
    mavenCentral()
}
dependencies {
    implementation("com.github.massive-com:client-jvm:v5.1.2")
    implementation("io.github.cdimascio:dotenv-kotlin:6.4.1")
}
```

**Key method examples:**
```kotlin
// Aggregates
val result = client.getAggregatesBlocking(AggregatesParameters(
    ticker = "AAPL", multiplier = 1, timespan = "day",
    fromDate = "2025-01-01", toDate = "2025-06-01", sort = "asc", limit = 30
))
result.results?.forEach { bar ->
    println("${bar.open} ${bar.high} ${bar.low} ${bar.close} ${bar.volume} ${bar.timestampMillis}")
}

// Last trade
val lastTrade = client.getLastTradeBlockingV2("AAPL")
val price = lastTrade.results?.price
```

**SDK package is `io.polygon.kotlin.sdk`.** Bar fields use full names: `open`, `high`, `low`, `close`, `volume`, `timestampMillis` (nullable Long, Unix ms). Results are nullable: `result.results?.forEach { ... }`.

**Timestamps:** `timestampMillis` is Unix milliseconds. Convert with `Instant.ofEpochMilli(bar.timestampMillis!!).atZone(ZoneOffset.UTC)`.

**Pagination:** Manual cursor-based via `nextUrl` on the response. No auto-pagination.

**WebSocket** (package `io.polygon.kotlin.sdk.websocket`):
```kotlin
val client = PolygonWebSocketClient(
    apiKey = env["MASSIVE_API_KEY"],
    feed = Feed.Delayed,
    market = Market.Stocks,
    listener = object : DefaultPolygonWebSocketListener() {
        override fun onAuthenticated(client: PolygonWebSocketClient) {
            client.subscribeBlocking(listOf(
                PolygonWebSocketSubscription(PolygonWebSocketChannel.Stocks.Trades, "AAPL")
            ))
        }
        override fun onReceive(client: PolygonWebSocketClient, message: PolygonWebSocketMessage) {
            when (message) {
                is PolygonWebSocketMessage.StocksMessage.Trade -> println("${message.price}")
                else -> {}
            }
        }
    }
)
client.connectBlocking()
```
Subscribe after `onAuthenticated` fires. Three connect variants: `connect()` (suspend), `connectBlocking()`, `connectAsync(callback)`.

## SDK method name cross-reference

| Operation | Python | JS/TS | Go | Kotlin |
|---|---|---|---|---|
| Stock aggregates | `list_aggs` | `getStocksAggregates({stocksTicker, ...})` | `GetStocksAggregatesWithResponse` | `getAggregatesBlocking(AggregatesParameters(...))` |
| Universal snapshot | `list_universal_snapshots` | `getSnapshots({tickerAnyOf})` | `GetSnapshotsWithResponse` | `getSnapshotAllTickersBlocking(...)` |
| Options chain | `list_snapshot_options_chain` | `getOptionsChain({underlyingAsset, ...})` | `GetOptionsChainWithResponse` | `getSnapshotOptionsChainBlocking(...)` |
| Last trade | `get_last_trade` | `getLastStocksTrade({stocksTicker})` | `GetLastStocksTradeWithResponse` | `getLastTradeBlockingV2(ticker)` |
| Last quote | `get_last_quote` | `getLastStocksQuote({stocksTicker})` | `GetLastStocksQuoteWithResponse` | `getLastQuoteBlockingV2(ticker)` |
| SMA | `get_sma` | `getStocksSMA({stockTicker, ...})` | `GetStocksSMAWithResponse` | `getSMABlocking(...)` |
| Market status | `get_market_status` | `getMarketStatus()` | `GetMarketStatusWithResponse` | `getMarketStatusBlocking()` |

## REST API coverage

The REST API spans these categories. Use the MCP `search_endpoints` tool to find specific endpoints.

**Market data (all asset classes):** aggregates/bars (custom timespan, daily summary, previous day), snapshots (single ticker, full market, top movers, unified cross-asset), technical indicators (SMA, EMA, RSI, MACD), trades, quotes, last trade/quote.

**Equities:** corporate actions (splits, dividends, IPOs, ticker events), fundamentals (balance sheets, income statements, cash flow statements, financial ratios, float, short interest, short volume), SEC filings (10-K sections, 8-K text, risk factors), news, related tickers.

**Options:** contract reference data, options chain snapshot with Greeks/IV, individual contract snapshots.

**Futures:** contract reference data, products, trading schedules, contract snapshots, trades, quotes.

**Economy/Federal Reserve:** treasury yields, inflation (CPI, PCE), inflation expectations, labor market indicators.

**Partner data ($99/mo per dataset, separate add-ons):** Benzinga (news, analyst ratings, earnings, corporate guidance, bull/bear cases, consensus ratings), ETF Global (analytics, constituents, fund flows, profiles), TMX/Wall Street Horizon (corporate events).

**Alternative data:** European consumer spending (merchant aggregates, merchant hierarchy).

**Reference data:** ticker search/overview, exchanges, market holidays, market status, condition codes.

## WebSocket streaming

```python
from massive.websocket.models import Market, Feed

ws = WebSocketClient(api_key=key, market=Market.Stocks, subscriptions=["T.AAPL"])
ws.run(handle_msg=lambda msgs: [print(m.price, m.size) for m in msgs])

# For delayed data (Starter plan):
ws = WebSocketClient(api_key=key, feed=Feed.Delayed, market=Market.Stocks, subscriptions=["T.AAPL"])

# For business feeds:
ws = WebSocketClient(api_key=key, feed=Feed.Business, market=Market.Stocks, subscriptions=["T.AAPL"])
```

**Market types:** `Market.Stocks`, `Market.Options`, `Market.Crypto`, `Market.Forex`, `Market.Indices`, `Market.Futures` (also `Market.FuturesCME`, `Market.FuturesCBOT`, `Market.FuturesNYMEX`, `Market.FuturesCOMEX`)

**Feed types:** `Feed.RealTime` (default), `Feed.Delayed`, `Feed.Business`, and others. The feed must match your plan tier.

**Subscription prefixes:**
- `T.` trades, `Q.` quotes, `A.` per-minute aggregates, `AS.` per-second aggregates
- `FMV.` fair market value (crypto, forex, options, stocks)
- `NOI.` net order imbalance (stocks only), `LULD.` limit up/limit down (stocks only)
- `V.` index values (indices only)
- Use `T.*` to subscribe to all tickers for a given type

**Flat files (S3):** Endpoint at `https://files.massive.com`, bucket `flatfiles`. Requires separate S3 credentials from [Dashboard > Keys](https://massive.com/dashboard/keys) (different from REST API key). Two env var naming conventions exist in examples: `MASSIVE_S3_ACCESS_KEY`/`MASSIVE_S3_SECRET_KEY` or `MASSIVE_FLATFILES_ACCESS_KEY_ID`/`MASSIVE_FLATFILES_SECRET_ACCESS_KEY`. Use `boto3` with `endpoint_url`. Files are gzipped CSV. Path pattern example: `us_stocks_sip/day_aggs_v1/`. Available for stocks, options, crypto, forex, and indices (day/minute aggregates, trades, quotes). Files have a ~1 day publication delay. Requires Starter plan or above.

## Plans and data access

Plans are **per asset class** (Stocks, Options, Indices, Currencies, Futures) and split into two categories:

**Individual plans** (personal, non-commercial use):
- **Basic** ($0/mo): 5 API calls/min, end-of-day data, 2yr history, reference data, corporate actions, technical indicators, minute aggregates. Good for testing.
- **Starter** ($29-49/mo): Unlimited calls, 15-min delayed data (10-min for futures), 2-5yr history. Adds flat files, WebSockets, snapshots, second aggregates. Options Starter adds real-time Greeks/IV and daily open interest.
- **Developer** ($79/mo): Unlimited calls, 15-min delayed, 4-10yr history. Adds trades.
- **Advanced** ($99-199/mo): Unlimited calls, **real-time data**, 5-20+yr history. Adds quotes. Stocks Advanced adds financials and ratios.

**Business plans** (commercial use, building products):
- **Stocks Business** ($1,999/mo): Everything including real-time FMV, no exchange fees, 20+yr history, financials and ratios, historical trades and quotes.
- **Options Business** ($1,999/mo): Everything including real-time FMV, 10+yr historical trades, 2.5yr historical quotes.
- **Indices Business** ($2,500/mo): Tickers licensed by feed, exchange assistance.
- **Currencies Business** ($999/mo): All forex and crypto tickers, full real-time access.
- **Futures Business** ($999/mo per exchange): Per exchange (CME, CBOT, NYMEX, COMEX).
- **Enterprise** (custom): Everything in Business plus SLAs, dedicated Slack channel, implementation support.

**Partner data** (separate add-ons, $99/mo per dataset for individual, pricing on request for business):
- Benzinga: Analyst Insights, Analyst Ratings, Bulls Bears Say, Corporate Guidance, Earnings, News (6 datasets).
- ETF Global: Analysis, Constituents, Fund Flows, Profiles and Exposure, Taxonomies (5 datasets).
- TMX: Corporate Events (1 dataset).

**Alternative data:** European Consumer Spending by Merchant ($99/mo, individual use only, 30-day data embargo).

Save 20% with annual billing. Futures individual plans (except Basic) are "Coming soon."

**Key question for developers:** If building a personal project or prototype, Individual plans work. If building a commercial product, SaaS, or redistributing data, Business plans are required.

## Common gotchas

- REST endpoints are **polled**, not streamed. Only WebSocket provides real-time streaming.
- Futures aggregates use a different endpoint (`/futures/vX/aggs/{ticker}`) than equity aggregates. The API design pattern is consistent, but the endpoints differ. Never claim "one endpoint" covers all asset classes.
- Always pass `sort="asc"` to `list_aggs()` for chronological data.
- Aggregates are split-adjusted by default (`adjusted=True`).
- Options filtering by `expiration_date` requires ISO format (`YYYY-MM-DD`).
- Individual tickers (e.g., Bitcoin, AAPL) are not "markets." Markets are asset classes (equities, crypto, forex, etc.).
- Flat files have a ~1 day publication delay. Files for yesterday may not be available yet.
- Free tier (Basic) is limited to 5 API calls/minute and end-of-day data only. Use TTL-based caching to avoid unnecessary calls.
- The SDK `limit` parameter controls page size (max 50,000 for aggregates), not total results. The client auto-paginates by default.
- WebSockets, flat files, and snapshots require Starter plan or above.
- Trades require Developer plan or above. Quotes require Advanced or above.
- Financials and ratios require Stocks Advanced ($199/mo) or Stocks Business.
- Real-time data requires Advanced or Business plan. Lower tiers get 15-min delayed data.
- Fair Market Value (FMV) is Business plans only.

## Project structure

All projects follow the same pattern: dependency file, entry point, `.env.example`, `.gitignore`, and README.

| Language | Dependency file | Entry point | Run command |
|---|---|---|---|
| Python | `pyproject.toml` (pin `massive>=2.4.0`) | `main.py` | `uv sync && uv run python main.py` |
| JS/TS | `package.json` (`@massive.com/client-js`) | `index.js` / `index.ts` | `npm install && node index.js` |
| Go | `go.mod` (`client-go/v3`) | `main.go` | `go mod tidy && go run main.go` |
| Kotlin | `build.gradle.kts` (`client-jvm:v5.1.2` from JitPack) | `src/main/kotlin/Main.kt` | `./gradlew run` |

## MCP server workflow

The Massive MCP server exposes four composable tools. Use them in this order:

1. `search_endpoints(query, scope)` - find relevant endpoints by natural language query. `scope`: `endpoints`, `functions`, or `all`.
2. `get_endpoint_docs(docs_url)` - get full parameter docs for a specific endpoint (use the `docs_url` from search results).
3. `call_api(endpoint, params, store_as, apply)` - call any REST endpoint; use `store_as` to save results as a DataFrame.
4. `query_data(sql, apply)` - run SQL (SQLite) against stored DataFrames.

Built-in financial functions available via `apply` parameter on `call_api` and `query_data`: Black-Scholes (`bs_price`, `bs_delta`, `bs_gamma`, `bs_theta`, `bs_vega`, `bs_rho`), returns (`simple_return`, `log_return`, `cumulative_return`, `sharpe_ratio`, `sortino_ratio`), technicals (`sma`, `ema`).

## Documentation

- Full REST endpoint reference: https://massive.com/docs/rest/llms-full.txt
- Top-level docs index: https://massive.com/docs/llms.txt
- Python SDK: `pip install massive` / https://pypi.org/project/massive/
- All SDKs and examples: https://github.com/massive-com
- Community examples and demos: https://github.com/massive-com/community
- MCP server: https://github.com/massive-com/mcp_massive
