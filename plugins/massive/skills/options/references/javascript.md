# JavaScript / TypeScript SDK pattern for options chain

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
