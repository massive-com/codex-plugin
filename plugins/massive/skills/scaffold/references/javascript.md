# JavaScript / TypeScript scaffold

Dependencies: `package.json`

```json
{
  "name": "$0",
  "version": "0.1.0",
  "type": "module",
  "dependencies": {
    "@massive.com/client-js": "^10.6.0",
    "dotenv": "^16.0.0"
  }
}
```

For TypeScript, add `"devDependencies": { "tsx": "^4.19.0", "@types/node": "^20.0.0" }` and create a `tsconfig.json`.

Entry point: `index.js` (or `index.ts`)

Run command: `npm install && node index.js` (or `npx tsx index.ts`)

`.gitignore` additions: `node_modules/`

## rest index.js

```javascript
import "dotenv/config";
import { restClient } from "@massive.com/client-js";

const client = restClient(process.env.MASSIVE_API_KEY);

const response = await client.getStocksAggregates({
  stocksTicker: "AAPL",
  multiplier: 1,
  timespan: "day",
  from: "2025-01-01",
  to: "2025-06-01",
  adjusted: true,
  sort: "asc",
  limit: 30,
});
for (const bar of response.results ?? []) {
  const dt = new Date(bar.t);
  console.log(`${dt.toISOString().slice(0, 10)}  O:${bar.o}  H:${bar.h}  L:${bar.l}  C:${bar.c}  V:${bar.v}`);
}
```

ALL SDK methods take a single object parameter with named fields. Bar fields are abbreviated: `o` (open), `h` (high), `l` (low), `c` (close), `v` (volume), `t` (Unix ms timestamp). Convert with `new Date(bar.t)`.

Pagination: pass `{ pagination: true }` as third arg to `restClient()` to auto-follow `next_url`. Without it, you get a single page.

## websocket index.js

```javascript
import "dotenv/config";
import { websocketClient } from "@massive.com/client-js";

const ws = websocketClient(process.env.MASSIVE_API_KEY, "wss://delayed.massive.com");
const stocksWS = ws.stocks();

stocksWS.onopen = () => {
  console.log("connected, subscribing to T.AAPL");
  stocksWS.send(JSON.stringify({ action: "subscribe", params: "T.AAPL" }));
};

stocksWS.onclose = (e) => console.log(`closed: ${e.code} ${e.reason}`);
stocksWS.onerror = (e) => console.error("error:", e.message ?? e);

stocksWS.onmessage = ({ data }) => {
  const messages = JSON.parse(data);
  for (const msg of messages) {
    if (msg.ev === "status") {
      console.log(`[status] ${msg.status}: ${msg.message}`);
    } else if (msg.ev === "T") {
      console.log(`Trade: ${msg.p} x ${msg.s}`);
    } else {
      console.log(msg);
    }
  }
};
```

Always log `status` events plus `onclose`/`onerror` — without them, auth failures, plan-tier rejections, and closed connections look like silent hangs. WebSocket returns W3CWebSocket instances. Message events: `T` (trade), `Q` (quote), `A` (per-second agg), `AM` (per-minute agg). Market connections: `ws.stocks()`, `ws.options()`, `ws.crypto()`, `ws.forex()`, `ws.indices()`, `ws.futures()`.
