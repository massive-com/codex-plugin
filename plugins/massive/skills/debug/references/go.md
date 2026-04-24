# Go-specific Massive errors

## `panic: missing API key`

`rest.New("")` panics if `MASSIVE_API_KEY` is not in the environment. Call `godotenv.Load()` first to load `.env`, or pass the key explicitly: `rest.New("your_key")`.

## Nil pointer dereference on response

- Response fields are pointers. Always nil-check: `if resp.JSON200 != nil && resp.JSON200.Results != nil`.
- Results are `*[]T` (pointer to slice). Dereference with `*resp.JSON200.Results`.
- Greeks on options are a pointer: `if opt.Greeks != nil`.

## Wrong method pattern

- All REST methods use `{OperationName}WithResponse` pattern: `GetStocksAggregatesWithResponse`, `GetOptionsChainWithResponse`, `GetLastStocksTradeWithResponse`.
- First param is always `context.Context`.
- Optional params use pointer types with `rest.Ptr(value)` helper.

## Misleading field names on Last Trade response

In `GetLastStocksTradeResponse.JSON200.Results`, the generated Go struct names the trade price `BidPrice` and size `BidSize` (JSON-tagged `p` and `s`). These are NOT bid fields; they carry the actual trade price and size. The documentation comments clarify this, but the Go field names are misleading. Use them anyway:

```go
trade, _ := c.GetLastStocksTradeWithResponse(ctx, "AAPL")
r := *trade.JSON200.Results
price := r.BidPrice  // this is the TRADE price
size := r.BidSize    // *float64, nil-check before dereferencing
```

## Nested struct vs pointer-struct patterns on options chain

In `GetOptionsChainResponse.JSON200.Results[*]`:

- `Details` is a **struct** (value type). Access `opt.Details.Ticker` directly; do not nil-check.
- `Greeks` is a **pointer** (`*struct`). Nil-check before accessing: `if opt.Greeks != nil { delta := opt.Greeks.Delta }`.
- Fields *inside* Greeks (`Delta`, `Gamma`, `Theta`, `Vega`) are direct `float64` values (not pointers).
- `ImpliedVolatility` is `*float64`. Nil-check before dereferencing.
- `LastQuote` is a struct; `LastTrade` is a pointer struct.

## Type casting for enum params

Some params need explicit type casting: `(*gen.GetOptionsChainParamsContractType)(rest.Ptr("call"))`.

## Pagination

- Auto-pagination is on by default. Disable with `rest.NewWithOptions("", rest.WithPagination(false))`.
- For manual pagination, check `resp.JSON200.NextUrl` and make follow-up requests.

## WebSocket

- Import: `massivews "github.com/massive-com/client-go/v3/websocket"` and `"github.com/massive-com/client-go/v3/websocket/models"`.
- Channel-based: read from `c.Output()` (messages) and `c.Error()` (fatal errors).
- Topics: `massivews.StocksTrades`, `massivews.StocksQuotes`, etc.
- Type-switch on output: `case models.EquityTrade`, `case models.EquityQuote`, `case models.EquityAgg`.

## Rate-limit retry

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
