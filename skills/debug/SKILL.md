---
name: debug
description: Debug Massive API errors, unexpected responses, or SDK issues. Use when API calls return errors, data looks wrong, pagination isn't working, or the SDK behaves unexpectedly.
argument-hint: "[paste error message or describe the issue]"
disable-model-invocation: false
allowed-tools: Read Grep mcp__massive__search_endpoints mcp__massive__get_endpoint_docs Bash
---

# Debug Massive API issue

Read the user's code and error output, then diagnose the issue. Detect the language from the code and provide fixes in the same language.

## Common error patterns (all languages)

### HTTP 401 Unauthorized
- API key not set or invalid.
- **Python:** Is `MASSIVE_API_KEY` in `.env`? Is `load_dotenv()` called before `RESTClient()`?
- **JavaScript:** Is `import "dotenv/config"` at the top? Is `process.env.MASSIVE_API_KEY` passed to `restClient()`?
- **Go:** Is `godotenv.Load()` called before `rest.New("")`? The Go client panics if the key is empty and `MASSIVE_API_KEY` env var is not set.
- **Kotlin:** Is the API key passed to `PolygonRestClient(apiKey)` constructor?
- Test with: `curl -H "Authorization: Bearer YOUR_KEY" "https://api.massive.com/v3/snapshot?ticker.any_of=AAPL"`

### HTTP 403 Forbidden
- Endpoint requires a higher plan tier or add-on.
- Data access unlocks by tier: Basic (end-of-day, aggs, reference) < Starter (+ WebSockets, flat files, snapshots, second aggs) < Developer (+ trades) < Advanced (+ quotes, real-time data, financials/ratios for stocks).
- Partner data (Benzinga, ETF Global, TMX) requires separate add-on subscriptions ($99/mo per dataset for individual).
- Business use (commercial products, redistribution) requires Business plan ($999-2,500/mo per asset class).
- Fair Market Value (FMV) is Business plan only.
- Fix: Check plan at massive.com/dashboard. Suggest which plan tier is needed.

### HTTP 404 Not Found
- Wrong ticker format. Check prefix conventions:
  - Crypto needs `X:` prefix (`X:BTCUSD`, not `BTCUSD`)
  - Forex needs `C:` prefix (`C:EURUSD`, not `EURUSD`)
  - Indices need `I:` prefix (`I:SPX`, not `SPX`)
- Wrong endpoint URL or method name in SDK.
- Use `search_endpoints` to find the correct endpoint.

### HTTP 429 Too Many Requests / rate limited
- Basic plan is limited to **5 API calls/min**. Starter and above have unlimited calls, but bursts can still trigger throttling.
- Fix in priority order:
  1. **Cache reference data** (ticker lists, exchanges, market holidays) for 24 hours; they rarely change.
  2. **Batch via the universal snapshot endpoint** (`/v3/snapshot`) instead of one call per ticker.
  3. **Serialize loops, do not parallelize** on Basic tier.
  4. **Retry with exponential backoff** for transient 429s (see per-language examples below).
  5. **Upgrade** to Starter ($29-49/mo) for unlimited calls.
- On Basic, auto-pagination over large date ranges can silently exhaust the quota. Cap with `limit=50000` or a narrower date range, or switch to flat files (Starter+).

### Empty results (no error)
- Date range falls on a weekend or market holiday. Try a known trading day.
- Ticker is delisted or invalid.
- For flat files: files have a ~1 day publication delay.
- For options: check that `expiration_date` is in ISO format (`YYYY-MM-DD`).
- For aggregates: check `from`/`to` date ordering and that sort is set to ascending.

## Python-specific errors

### Pagination issues
- SDK `list_` methods return **generators**, not lists. You must iterate or wrap with `list()`.
- The `limit` parameter controls **page size**, not total results. The SDK auto-paginates by default. To cap total results: `list(islice(client.list_aggs(...), 1000))`.
- To disable auto-pagination: `RESTClient(api_key=key, pagination=False)`.
- Do not call `len()` on a generator. Convert to list first.

### `get_*` method misuse
- `get_*` methods (`get_last_trade`, `get_last_quote`, `get_market_status`, `get_sma`, `get_rsi`, `get_ema`, `get_macd`, etc.) return **single result objects**, not iterators.
- Do NOT wrap them in `list()` or `islice()`. This causes `TypeError: ... object is not iterable`.
- Technical indicators (`get_sma`, `get_rsi`, `get_ema`, `get_macd`) return a `SingleIndicatorResults` object. Access the data via `result.values` (list of objects with `.timestamp` and `.value`).

### Timestamp confusion
- Aggregates: millisecond epoch in `bar.timestamp`.
- Trades: nanosecond epoch in `trade.sip_timestamp`.
- Convert with `pd.to_datetime(ts, unit="ms", utc=True)` or `unit="ns"` for trades.
- Do NOT use strftime directly on `bar.timestamp` (it is an integer, not a datetime).

### WebSocket issues
- Check that `market` parameter matches the data type: `Market.Stocks` for equities, `Market.Options` for options, `Market.Crypto` for crypto.
- Import: `from massive.websocket.models import Market, Feed`
- Subscription format: `T.AAPL` for trades, `A.AAPL` for aggregates, `Q.AAPL` for quotes.
- `client.run()` blocks. Run in a daemon thread if combining with other logic.

### SDK version issues
- Ensure `massive>=2.4.0` in `pyproject.toml`. Check: `pip show massive` or `uv pip show massive`.

### Rate-limit retry (Python)
The SDK raises `massive.exceptions.BadResponse` on any non-200 response, passing the response body as the message (the HTTP status code is NOT preserved on the exception). So matching against `str(e)` looks at the JSON error body, which typically contains `"maximum requests"` or `"rate"` for 429s. Detect and retry:
```python
import time
from massive.exceptions import BadResponse

_RATE_LIMIT_MARKERS = ("maximum requests", "rate limit", "too many requests")

def _is_rate_limit(e: BadResponse) -> bool:
    msg = str(e).lower()
    return any(m in msg for m in _RATE_LIMIT_MARKERS)

def with_backoff(fn, max_attempts=5):
    for attempt in range(max_attempts):
        try:
            return fn()
        except BadResponse as e:
            if not _is_rate_limit(e) or attempt == max_attempts - 1:
                raise
            time.sleep(2 ** attempt)  # 1s, 2s, 4s, 8s, 16s

bars = with_backoff(lambda: list(client.list_aggs("AAPL", 1, "day", "2025-01-01", "2025-06-01")))
```
If you need strict HTTP-status-based retries, pass `raw=True` to the underlying REST call to get the `urllib3.HTTPResponse` and inspect `resp.status` directly.

For Streamlit, prefer `@st.cache_data(ttl=...)` over retries. Typical TTLs: 30s for snapshots, 300s for intraday aggregates, 86400s for reference data.

## JavaScript/TypeScript-specific errors

### Unhandled promise rejection / async errors
- All REST methods return Promises. Must use `await` or `.then()`. Wrap in `async function main()` with `.catch(console.error)`.
- Common mistake: calling `client.getStocksAggregates(...)` without `await` gives a Promise object, not data.

### `Cannot read properties of undefined (reading 'results')`
- The response object may not have `results` if the request failed. Always use optional chaining: `response.results ?? []`.
- Check `response.status` for error details.

### Wrong method names or calling convention
- JS SDK uses `getStocksAggregates` (not `list_aggs`), `getOptionsChain` (not `list_snapshot_options_chain`), `getLastStocksTrade` (not `get_last_trade`).
- ALL methods take a single object parameter: `client.getStocksAggregates({ stocksTicker: "AAPL", multiplier: 1, timespan: "day", from: "...", to: "..." })`. Do NOT use positional arguments.
- Technical indicator methods use `stockTicker` (no 's'): `getStocksSMA({ stockTicker: "AAPL", ... })`.
- `getSnapshots` takes `tickerAnyOf` as a comma-separated string, NOT an array.
- Bar fields are abbreviated: `o` (open), `h` (high), `l` (low), `c` (close), `v` (volume), `t` (timestamp in ms).

### Pagination
- Pagination is NOT iterator-based. Pass `{ pagination: true }` as the third arg to `restClient()` to auto-follow `next_url`. Without it, you get a single page.
- There is no `islice()` equivalent. Control results via the `limit` parameter.

### WebSocket
- WebSocket uses raw `send()` with JSON: `ws.send(JSON.stringify({ action: "subscribe", params: "T.AAPL" }))`.
- Market connections: `ws.stocks()`, `ws.options()`, `ws.crypto()`, etc. Each returns a W3C WebSocket.
- Message events: `T` (trade), `Q` (quote), `A` (per-second agg), `AM` (per-minute agg).

### Rate-limit retry (JavaScript/TypeScript)
SDK errors surface with a `status` field on HTTP failures. Retry 429s with exponential backoff:
```javascript
async function withBackoff(fn, maxAttempts = 5) {
  for (let i = 0; i < maxAttempts; i++) {
    try {
      return await fn();
    } catch (err) {
      const status = err?.status ?? err?.response?.status;
      if (status !== 429 || i === maxAttempts - 1) throw err;
      await new Promise(r => setTimeout(r, 2 ** i * 1000));
    }
  }
}

const resp = await withBackoff(() =>
  client.getStocksAggregates({ stocksTicker: "AAPL", multiplier: 1, timespan: "day", from: "2025-01-01", to: "2025-06-01" })
);
```

## Go-specific errors

### `panic: missing API key`
- `rest.New("")` panics if `MASSIVE_API_KEY` is not in the environment. Call `godotenv.Load()` first to load `.env`, or pass the key explicitly: `rest.New("your_key")`.

### Nil pointer dereference on response
- Response fields are pointers. Always nil-check: `if resp.JSON200 != nil && resp.JSON200.Results != nil`.
- Results are `*[]T` (pointer to slice). Dereference with `*resp.JSON200.Results`.
- Greeks on options are a pointer: `if opt.Greeks != nil`.

### Wrong method pattern
- All REST methods use `{OperationName}WithResponse` pattern: `GetStocksAggregatesWithResponse`, `GetOptionsChainWithResponse`, `GetLastStocksTradeWithResponse`.
- First param is always `context.Context`.
- Optional params use pointer types with `rest.Ptr(value)` helper.

### Misleading field names on Last Trade response
In `GetLastStocksTradeResponse.JSON200.Results`, the generated Go struct names the trade price `BidPrice` and size `BidSize` (JSON-tagged `p` and `s`). These are NOT bid fields; they carry the actual trade price and size. The documentation comments clarify this, but the Go field names are misleading. Use them anyway:
```go
trade, _ := c.GetLastStocksTradeWithResponse(ctx, "AAPL")
r := *trade.JSON200.Results
price := r.BidPrice  // this is the TRADE price
size := r.BidSize    // *float64, nil-check before dereferencing
```

### Nested struct vs pointer-struct patterns on options chain
In `GetOptionsChainResponse.JSON200.Results[*]`:
- `Details` is a **struct** (value type). Access `opt.Details.Ticker` directly; do not nil-check.
- `Greeks` is a **pointer** (`*struct`). Nil-check before accessing: `if opt.Greeks != nil { delta := opt.Greeks.Delta }`.
- Fields *inside* Greeks (`Delta`, `Gamma`, `Theta`, `Vega`) are direct `float64` values (not pointers).
- `ImpliedVolatility` is `*float64`. Nil-check before dereferencing.
- `LastQuote` is a struct; `LastTrade` is a pointer struct.

### Type casting for enum params
- Some params need explicit type casting: `(*gen.GetOptionsChainParamsContractType)(rest.Ptr("call"))`.

### Pagination
- Auto-pagination is on by default. Disable with `rest.NewWithOptions("", rest.WithPagination(false))`.
- For manual pagination, check `resp.JSON200.NextUrl` and make follow-up requests.

### WebSocket
- Import: `massivews "github.com/massive-com/client-go/v3/websocket"` and `"github.com/massive-com/client-go/v3/websocket/models"`.
- Channel-based: read from `c.Output()` (messages) and `c.Error()` (fatal errors).
- Topics: `massivews.StocksTrades`, `massivews.StocksQuotes`, etc.
- Type-switch on output: `case models.EquityTrade`, `case models.EquityQuote`, `case models.EquityAgg`.

### Rate-limit retry (Go)
429s come back as an error with "429" in the message (or check the response status). Retry with exponential backoff:
```go
import (
    "strings"
    "time"
)

func withBackoff[T any](fn func() (T, error), maxAttempts int) (T, error) {
    var zero T
    var lastErr error
    for i := 0; i < maxAttempts; i++ {
        result, err := fn()
        if err == nil {
            return result, nil
        }
        lastErr = err
        if !strings.Contains(err.Error(), "429") || i == maxAttempts-1 {
            return zero, err
        }
        time.Sleep(time.Duration(1<<i) * time.Second)
    }
    return zero, lastErr
}

resp, err := withBackoff(func() (*rest.GetStocksAggregatesResponse, error) {
    return c.GetStocksAggregatesWithResponse(ctx, "AAPL", 1, gen.Day, "2025-01-01", "2025-06-01", params)
}, 5)
```
Requires Go 1.21+ for generics. For older Go, write a type-specific wrapper.

## Kotlin-specific errors

### `NullPointerException` or missing API key
- Pass the API key directly to the `PolygonRestClient` constructor: `PolygonRestClient(apiKey)`.
- Use `dotenv-kotlin` to load `.env`: `val env = dotenv(); val client = PolygonRestClient(env["MASSIVE_API_KEY"])`.

### Null results
- `results` is nullable. Always use safe calls: `result.results?.forEach { ... }`.
- Greeks is nullable: `opt.greeks?.let { g -> ... }`.

### Wrong dependency coordinates
- The SDK is on JitPack (NOT Maven Central). Gradle dep: `implementation("com.github.massive-com:client-jvm:v5.1.2")`.
- Must add `maven("https://jitpack.io")` to repositories.

### Wrong class names or imports
- The SDK package is `io.polygon.kotlin.sdk`, not `org.openapitools` or `io.massive`.
- REST client: `io.polygon.kotlin.sdk.rest.PolygonRestClient`, not `DefaultApi`.
- Methods use `Blocking` suffix: `getAggregatesBlocking(AggregatesParameters(...))`, not `getStocksAggregates(...)`.
- Bar fields use full names: `bar.open`, `bar.high`, `bar.close`, `bar.volume`, `bar.timestampMillis` (not `bar.o`, `bar.h`, etc.).

### WebSocket
- Import from `io.polygon.kotlin.sdk.websocket.*`.
- Classes use `Polygon` prefix: `PolygonWebSocketClient`, `PolygonWebSocketChannel`, `PolygonWebSocketMessage`.
- Subscribe after `onAuthenticated` callback fires.
- Three connect variants: `connect()` (suspend), `connectBlocking()`, `connectAsync(callback)`.

### Rate-limit retry (Kotlin)
Wrap SDK calls with an exponential-backoff helper:
```kotlin
fun <T> withBackoff(maxAttempts: Int = 5, fn: () -> T): T {
    repeat(maxAttempts) { attempt ->
        try {
            return fn()
        } catch (e: Exception) {
            val isRateLimit = e.message?.contains("429") == true
            if (!isRateLimit || attempt == maxAttempts - 1) throw e
            Thread.sleep((1L shl attempt) * 1000)  // 1s, 2s, 4s, 8s, 16s
        }
    }
    error("unreachable")
}

val result = withBackoff { client.getAggregatesBlocking(params) }
```

## Diagnostic steps

1. Read the user's code and identify the failing call and language.
2. Match the error to a pattern above for that language.
3. If the error doesn't match a known pattern, use `search_endpoints` and `get_endpoint_docs` to verify the endpoint exists and check required parameters.
4. Suggest a specific fix with corrected code in the same language.
