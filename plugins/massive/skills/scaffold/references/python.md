# Python scaffold

Dependencies: `pyproject.toml`

```toml
[project]
name = "$0"
version = "0.1.0"
requires-python = ">=3.9"
dependencies = [
    "massive>=2.4.0",
    "python-dotenv>=1.0.0",
]
```

For **streamlit** type, add:

```toml
    "streamlit>=1.41.0",
    "plotly>=5.24.0",
    "pandas>=2.2.0",
```

Entry point: `main.py` (or `streamlit_app.py` for streamlit type)

Run command: `uv sync && uv run python main.py`

`.gitignore` additions: `.venv/`, `__pycache__/`, `.uv/`, `uv.lock`

## rest main.py

```python
from itertools import islice
from datetime import datetime, timezone
from dotenv import load_dotenv

load_dotenv()  # must come before importing RESTClient so env vars are set

from massive import RESTClient

client = RESTClient()  # reads MASSIVE_API_KEY from .env

bars = list(islice(client.list_aggs("AAPL", 1, "day", "2025-01-01", "2025-06-01", sort="asc"), 30))
for bar in bars:
    dt = datetime.fromtimestamp(bar.timestamp / 1000, tz=timezone.utc)
    print(f"{dt:%Y-%m-%d}  O:{bar.open:.2f}  H:{bar.high:.2f}  L:{bar.low:.2f}  C:{bar.close:.2f}  V:{bar.volume:.0f}")
```

Timestamps are millisecond epoch integers. Convert with `datetime.fromtimestamp(bar.timestamp / 1000)` or `pd.to_datetime(bar.timestamp, unit="ms", utc=True)`. Do NOT use strftime directly on bar.timestamp.

## websocket main.py

```python
import os
from dotenv import load_dotenv
from massive import WebSocketClient
from massive.websocket.models import Market, Feed

load_dotenv()

ws = WebSocketClient(
    api_key=os.environ["MASSIVE_API_KEY"],
    market=Market.Stocks,
    feed=Feed.Delayed,  # use Feed.RealTime for Advanced/Business plans
    subscriptions=["T.AAPL"],
)

def handle_msg(msgs):
    for m in msgs:
        if getattr(m, "event_type", None) == "status":
            print(f"[status] {getattr(m, 'status', '?')}: {getattr(m, 'message', '')}")
            continue
        if hasattr(m, "price") and hasattr(m, "size"):
            print(f"Trade: {m.price} x {m.size}")
        else:
            print(m)

ws.run(handle_msg=handle_msg)
```

Print status messages and fall through to `print(m)` for unrecognized types. The Massive WS protocol delivers connection and auth events as `status` messages — if `handle_msg` only handles trades, auth failures and empty feeds look like silent hangs. A user should see `auth_failed: ...` clearly, not wonder why nothing is printing.

## streamlit streamlit_app.py structure

1. `from dotenv import load_dotenv` and `load_dotenv()` at top
2. `@st.cache_resource` for client singleton
3. `@st.cache_data(ttl=30)` for API call wrappers
4. Sidebar for ticker input
5. A price chart using Plotly with `list_aggs()` data
6. Timestamps: `pd.to_datetime(bar.timestamp, unit="ms", utc=True)`
7. Run with `uv run streamlit run streamlit_app.py`
