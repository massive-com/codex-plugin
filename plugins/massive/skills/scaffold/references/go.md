# Go scaffold

Dependencies: `go.mod`

```
module $0

go 1.21

require github.com/massive-com/client-go/v3 latest
```

Also add `github.com/joho/godotenv` for .env loading.

Entry point: `main.go`

Run command: `go mod tidy && go run main.go`

`.gitignore` additions: none needed beyond `.env`

## rest main.go

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "github.com/joho/godotenv"
    "github.com/massive-com/client-go/v3/rest"
    "github.com/massive-com/client-go/v3/rest/gen"
)

func main() {
    godotenv.Load() // loads .env file
    c := rest.New("") // reads MASSIVE_API_KEY from env

    params := &gen.GetStocksAggregatesParams{
        Adjusted: rest.Ptr(true),
        Sort:     "asc",
        Limit:    rest.Ptr(30),
    }

    resp, err := c.GetStocksAggregatesWithResponse(
        context.Background(), "AAPL", 1, gen.Day, "2025-01-01", "2025-06-01", params,
    )
    if err != nil {
        log.Fatal(err)
    }

    if resp.JSON200 != nil && resp.JSON200.Results != nil {
        for _, bar := range *resp.JSON200.Results {
            dt := time.UnixMilli(int64(bar.Timestamp))
            fmt.Printf("%s  O:%.2f  H:%.2f  L:%.2f  C:%.2f  V:%.0f\n",
                dt.UTC().Format("2006-01-02"), bar.O, bar.H, bar.L, bar.C, bar.V)
        }
    }
}
```

Bar fields: `O`, `H`, `L`, `C`, `V`, `Timestamp` (Unix ms). Pointer helpers: `rest.Ptr(value)` for optional params.

All generated methods follow the pattern `{OperationName}WithResponse`. Response data is in `resp.JSON200`. Results are a pointer to a slice (`*[]T`).

Pagination: auto-pagination is on by default. Disable with `rest.NewWithOptions("", rest.WithPagination(false))`.

## websocket main.go

```go
package main

import (
    "fmt"
    "os"
    "os/signal"

    "github.com/joho/godotenv"
    massivews "github.com/massive-com/client-go/v3/websocket"
    "github.com/massive-com/client-go/v3/websocket/models"
)

func main() {
    godotenv.Load()

    c, err := massivews.New(massivews.Config{
        APIKey: os.Getenv("MASSIVE_API_KEY"),
        Feed:   massivews.Delayed,
        Market: massivews.Stocks,
    })
    if err != nil {
        panic(err)
    }
    defer c.Close()

    c.Subscribe(massivews.StocksTrades, "AAPL")
    if err := c.Connect(); err != nil {
        panic(err)
    }

    sigint := make(chan os.Signal, 1)
    signal.Notify(sigint, os.Interrupt)

    for {
        select {
        case <-sigint:
            return
        case <-c.Error():
            return
        case out, more := <-c.Output():
            if !more {
                return
            }
            switch t := out.(type) {
            case models.EquityTrade:
                fmt.Printf("Trade: %.2f x %d\n", t.Price, t.Size)
            }
        }
    }
}
```

WebSocket uses channels: `c.Output()` for messages, `c.Error()` for fatal errors. Topics: `massivews.StocksTrades`, `massivews.StocksQuotes`, `massivews.StocksMinAggs`, etc.
