# Python SDK pattern for options chain

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
