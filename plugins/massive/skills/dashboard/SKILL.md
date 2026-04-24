---
name: massive-dashboard
description: Scaffold a Streamlit-based financial dashboard using Massive APIs. Use when building Massive-backed market data visualizations, multi-asset dashboards, or monitoring interfaces. Do not use for dashboards unrelated to Massive.
---

# Scaffold a Streamlit financial dashboard

Project name: $0 (default: `dashboard` if not specified)
Focus area: $1 (default: `multi-asset` if not specified)

Prefer SDK-backed data access and cached helper functions. Do not add raw HTTP calls when the Massive SDK already covers the workflow.

**Keep the app minimal.** One file, direct `RESTClient()` calls wrapped in `@st.cache_data`, straightforward Plotly charts. Do not add env-var configuration layers, plugin systems, or utility modules (`require_env`, `value_from`, `first_available`, format helpers) beyond what Streamlit natively needs. A working 80-line `streamlit_app.py` beats a 300-line multi-module dashboard.

## Brand rules (apply to ALL generated files)

- No emojis anywhere, including Streamlit `page_icon`. Use a text string or None instead.
- No em dashes. Use commas, periods, semicolons, or parentheses. For NA/missing values in tables, use "N/A" or "-", not an em dash.
- The only valid API domain is `api.massive.com`. Never use any other API domain in generated code.

## Architecture

**Default layout — single file:**

```
$0/
  streamlit_app.py       # Everything: client, caching, sidebar, charts
  pyproject.toml
  .env.example
  README.md
```

Start here. Put `@st.cache_resource` client, `@st.cache_data` API wrappers, sidebar controls, and Plotly charts all in `streamlit_app.py`. Expect ~80-150 lines for most focus areas.

**Multi-module layout — only when the user explicitly asks for a "terminal," "Bloomberg-style," or "multi-panel" app.** In that case (and only that case), split into:

```
$0/
  streamlit_app.py       # Entry point: layout, styling, sidebar
  terminal/
    data.py              # API calls + caching
    config.py            # Colors, TTL constants
    charts.py            # Chart rendering functions
    panels/              # One file per panel
  .streamlit/
    config.toml          # Theme configuration
  pyproject.toml
  .env.example
  README.md
```

Do not introduce the multi-module layout for a plain "dashboard" or any focus area the user didn't describe as terminal-grade.

## Key patterns

### Client singleton (never recreate per call)
```python
import streamlit as st

@st.cache_resource
def _get_client(api_key: str):
    from massive import RESTClient
    return RESTClient(api_key=api_key)
```

### TTL-based caching (reduce API calls)
```python
TTL_SNAPSHOT = 30     # live quotes refresh every 30s
TTL_CHART = 60        # historical bars refresh every 60s
TTL_OPTIONS = 45      # options chain refresh every 45s
TTL_NEWS = 120        # news refresh every 2 min
TTL_MACRO = 3600      # macro data refresh every hour

@st.cache_data(ttl=TTL_SNAPSHOT, show_spinner=False)
def get_snapshot(api_key: str, tickers: tuple[str, ...]) -> dict:
    client = _get_client(api_key)
    return {s.ticker: s for s in client.list_universal_snapshots(ticker_any_of=list(tickers))}

@st.cache_data(ttl=TTL_CHART, show_spinner=False)
def get_aggs(api_key: str, ticker: str, multiplier: int, timespan: str, lookback_days: int) -> list:
    client = _get_client(api_key)
    from datetime import datetime, timedelta
    from itertools import islice
    to_date = datetime.now().strftime("%Y-%m-%d")
    from_date = (datetime.now() - timedelta(days=lookback_days)).strftime("%Y-%m-%d")
    return list(islice(client.list_aggs(ticker, multiplier, timespan, from_date, to_date, sort="asc"), 5000))
```
Always pass `sort="asc"` to `list_aggs()` for chronological data.

Note: pass `api_key` as a plain string and ticker lists as tuples so they are hashable for `st.cache_data`.

### Plotly dark theme charts
```python
import plotly.graph_objects as go

fig = go.Figure(data=[go.Candlestick(
    x=dates, open=opens, high=highs, low=lows, close=closes
)])
fig.update_layout(
    template="plotly_dark",
    paper_bgcolor="#000000",
    plot_bgcolor="#0a0a0a",
    xaxis_rangeslider_visible=False,
)
```

### Technical indicators (SMA, EMA, RSI, MACD)
These methods return a `SingleIndicatorResults` object, NOT a paginated iterator. Access `.values` to get the list of data points:
```python
@st.cache_data(ttl=TTL_CHART, show_spinner=False)
def get_rsi(api_key: str, ticker: str, window: int, timespan: str) -> list:
    client = _get_client(api_key)
    result = client.get_rsi(ticker, params={"window": window, "timespan": timespan, "sort": "asc"})
    return result.values if result.values else []
```
Each item has `.timestamp` (ms epoch) and `.value`. Do NOT wrap these calls in `list(islice(...))`.

### Multi-asset watchlist
Use `list_universal_snapshots()` with mixed-asset tickers in a single call:
```python
tickers = ("AAPL", "MSFT", "X:BTCUSD", "C:EURUSD", "I:SPX")
snapshots = get_snapshot(api_key, tickers)
```

### WebSocket trade tape (real-time panel)
Run WebSocket in a daemon thread; append messages to a session-scoped buffer:
```python
if "trade_buffer" not in st.session_state:
    st.session_state.trade_buffer = []

# Start WebSocket thread (see bloomberg-terminal/terminal/panels/trade_tape.py)
```

## Focus areas

### multi-asset (default)
Panels: watchlist (equities + crypto + forex + indices), price chart with indicators, market status.
SDK: `list_universal_snapshots`, `list_aggs`, `get_sma`/`get_ema`/`get_rsi`/`get_macd`, `get_market_status()`.
For market status, use `client.get_market_status()` from the SDK. Do NOT make raw REST calls to any domain other than `api.massive.com`.

### options
Panels: options chain table, Greeks heatmap, P&L diagram, underlying price chart.
SDK: `list_snapshot_options_chain`, `list_aggs`, `get_last_trade`.

### crypto
Panels: crypto watchlist, BTC/ETH price charts, volume comparison.
SDK: `list_universal_snapshots` with `X:` tickers, `list_aggs`.

### macro
Panels: treasury yield curve, inflation chart, labor market indicators, fed funds rate.
SDK: `list_treasury_yields()`, `list_inflation()`, `list_labor_market_indicators()`, `get_market_status()`.
Do NOT use the `requests` library or any raw HTTP calls. Use only the Massive Python SDK for all API calls. Do NOT add `requests` to `pyproject.toml`. If an SDK method is not available for a specific endpoint, note it in a comment but do not fall back to raw HTTP.
Note: Economy/Federal Reserve data may require a specific plan tier. Check access at massive.com/dashboard.

## Dependencies

```toml
[project]
name = "$0"
version = "0.1.0"
requires-python = ">=3.9"
dependencies = [
    "massive>=2.4.0",
    "streamlit>=1.41.0",
    "plotly>=5.24.0",
    "pandas>=2.2.0",
    "numpy>=2.0.0",
    "python-dotenv>=1.0.0",
]
```

## Streamlit config

`.streamlit/config.toml`:
```toml
[theme]
base = "dark"

[server]
headless = true
```

## Steps

1. Create the project directory structure
2. Write `config.py` with color palette and TTL constants
3. Write `data.py` with cached API call wrappers for the chosen focus area
4. Write `charts.py` with Plotly rendering functions
5. Write panel modules for the chosen focus area
6. Write `streamlit_app.py` with layout, sidebar, and panel composition
7. Write `pyproject.toml`, `.env.example`, `.gitignore`, `.streamlit/config.toml`, README
8. Provide quickstart:
   ```
   cd $0
   cp .env.example .env
   # Add your Massive API key to .env
   uv sync
   uv run streamlit run streamlit_app.py
   ```
9. Note: Dashboards use Streamlit (Python only). The user will need at least a Starter plan for snapshots and WebSocket data.
