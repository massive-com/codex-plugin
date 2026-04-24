# Python-specific Massive errors

## Pagination issues

- SDK `list_` methods return **generators**, not lists. You must iterate or wrap with `list()`.
- The `limit` parameter controls **page size**, not total results. The SDK auto-paginates by default. To cap total results: `list(islice(client.list_aggs(...), 1000))`.
- To disable auto-pagination: `RESTClient(api_key=key, pagination=False)`.
- Do not call `len()` on a generator. Convert to list first.

## `get_*` method misuse

- `get_*` methods (`get_last_trade`, `get_last_quote`, `get_market_status`, `get_sma`, `get_rsi`, `get_ema`, `get_macd`, etc.) return **single result objects**, not iterators.
- Do NOT wrap them in `list()` or `islice()`. This causes `TypeError: ... object is not iterable`.
- Technical indicators (`get_sma`, `get_rsi`, `get_ema`, `get_macd`) return a `SingleIndicatorResults` object. Access the data via `result.values` (list of objects with `.timestamp` and `.value`).

## Timestamp confusion

- Aggregates: millisecond epoch in `bar.timestamp`.
- Trades: nanosecond epoch in `trade.sip_timestamp`.
- Convert with `pd.to_datetime(ts, unit="ms", utc=True)` or `unit="ns"` for trades.
- Do NOT use strftime directly on `bar.timestamp` (it is an integer, not a datetime).

## WebSocket issues

- Check that `market` parameter matches the data type: `Market.Stocks` for equities, `Market.Options` for options, `Market.Crypto` for crypto.
- Import: `from massive.websocket.models import Market, Feed`
- Subscription format: `T.AAPL` for trades, `A.AAPL` for aggregates, `Q.AAPL` for quotes.
- `client.run()` blocks. Run in a daemon thread if combining with other logic.

## SDK version issues

Ensure `massive>=2.4.0` in `pyproject.toml`. Check: `pip show massive` or `uv pip show massive`.

## Rate-limit retry

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
