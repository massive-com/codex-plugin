# JavaScript / TypeScript-specific Massive errors

## Unhandled promise rejection / async errors

- All REST methods return Promises. Must use `await` or `.then()`. Wrap in `async function main()` with `.catch(console.error)`.
- Common mistake: calling `client.getStocksAggregates(...)` without `await` gives a Promise object, not data.

## `Cannot read properties of undefined (reading 'results')`

- The response object may not have `results` if the request failed. Always use optional chaining: `response.results ?? []`.
- Check `response.status` for error details.

## Wrong method names or calling convention

- JS SDK uses `getStocksAggregates` (not `list_aggs`), `getOptionsChain` (not `list_snapshot_options_chain`), `getLastStocksTrade` (not `get_last_trade`).
- ALL methods take a single object parameter: `client.getStocksAggregates({ stocksTicker: "AAPL", multiplier: 1, timespan: "day", from: "...", to: "..." })`. Do NOT use positional arguments.
- Technical indicator methods use `stockTicker` (no 's'): `getStocksSMA({ stockTicker: "AAPL", ... })`.
- `getSnapshots` takes `tickerAnyOf` as a comma-separated string, NOT an array.
- Bar fields are abbreviated: `o` (open), `h` (high), `l` (low), `c` (close), `v` (volume), `t` (timestamp in ms).

## Pagination

- Pagination is NOT iterator-based. Pass `{ pagination: true }` as the third arg to `restClient()` to auto-follow `next_url`. Without it, you get a single page.
- There is no `islice()` equivalent. Control results via the `limit` parameter.

## WebSocket

- WebSocket uses raw `send()` with JSON: `ws.send(JSON.stringify({ action: "subscribe", params: "T.AAPL" }))`.
- Market connections: `ws.stocks()`, `ws.options()`, `ws.crypto()`, etc. Each returns a W3C WebSocket.
- Message events: `T` (trade), `Q` (quote), `A` (per-second agg), `AM` (per-minute agg).

## Rate-limit retry

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
