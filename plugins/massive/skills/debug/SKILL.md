---
name: massive-debug
description: Debug Massive API errors, unexpected responses, or SDK issues. Use when the failing code, HTTP error, or confusing behavior involves Massive. Do not use for generic app bugs unrelated to Massive.
---

# Debug Massive API issue

Read the user's code and error output, then diagnose the issue. Detect the language from the code and provide fixes in the same language.

When current endpoint names, parameters, or plan access details matter, prefer the bundled Massive MCP tools over memory. If those tools are unavailable, say current verification is unavailable and avoid inventing exact endpoint details.

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
- Use `search_endpoints` to find the correct endpoint. If the tool is unavailable, say you cannot verify the current endpoint name and focus on the SDK-side fix instead of guessing.

### HTTP 429 Too Many Requests / rate limited

- Basic plan is limited to **5 API calls/min**. Starter and above have unlimited calls, but bursts can still trigger throttling.
- Fix in priority order:
  1. **Cache reference data** (ticker lists, exchanges, market holidays) for 24 hours; they rarely change.
  2. **Batch via the universal snapshot endpoint** (`/v3/snapshot`) instead of one call per ticker.
  3. **Serialize loops, do not parallelize** on Basic tier.
  4. **Retry with exponential backoff** for transient 429s — see the per-language reference for a working helper.
  5. **Upgrade** to Starter ($29-49/mo) for unlimited calls.
- On Basic, auto-pagination over large date ranges can silently exhaust the quota. Cap with `limit=50000` or a narrower date range, or switch to flat files (Starter+).

### Empty results (no error)

- Date range falls on a weekend or market holiday. Try a known trading day.
- Ticker is delisted or invalid.
- For flat files: files have a ~1 day publication delay.
- For options: check that `expiration_date` is in ISO format (`YYYY-MM-DD`).
- For aggregates: check `from`/`to` date ordering and that sort is set to ascending.

## Language-specific errors

After identifying the language, read the matching reference for SDK-specific pitfalls, pagination, WebSocket shapes, and a rate-limit retry helper:

- **Python** → [references/python.md](references/python.md)
- **JavaScript / TypeScript** → [references/javascript.md](references/javascript.md)
- **Go** → [references/go.md](references/go.md)
- **Kotlin / JVM** → [references/kotlin.md](references/kotlin.md)

Only read the reference for the detected language.

## Diagnostic steps

1. Read the user's code and identify the failing call and language.
2. Match the error to a common pattern above, then consult the language reference for SDK-specific details.
3. If the error doesn't match a known pattern, use `search_endpoints` and `get_endpoint_docs` to verify the endpoint exists and check required parameters.
4. Suggest a specific fix with corrected code in the same language.
