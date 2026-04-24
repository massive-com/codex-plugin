# Go SDK pattern for options chain

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/joho/godotenv"
    "github.com/massive-com/client-go/v3/rest"
    "github.com/massive-com/client-go/v3/rest/gen"
)

func main() {
    godotenv.Load()
    c := rest.New("")

    // Get current price
    tradeResp, err := c.GetLastStocksTradeWithResponse(context.Background(), "SPY")
    if err != nil {
        log.Fatal(err)
    }
    // spot price: tradeResp.JSON200.Results fields

    // Fetch options chain
    params := &gen.GetOptionsChainParams{
        StrikePriceGte:    rest.Ptr(float32(500)),
        StrikePriceLte:    rest.Ptr(float32(600)),
        ExpirationDateGte: rest.Ptr("2025-06-01"),
        ExpirationDateLte: rest.Ptr("2025-06-30"),
        ContractType:      (*gen.GetOptionsChainParamsContractType)(rest.Ptr("call")),
        Limit:             rest.Ptr(250),
        Order:             (*gen.GetOptionsChainParamsOrder)(rest.Ptr("asc")),
    }

    resp, err := c.GetOptionsChainWithResponse(context.Background(), "SPY", params)
    if err != nil {
        log.Fatal(err)
    }

    if resp.JSON200 != nil && resp.JSON200.Results != nil {
        for _, opt := range *resp.JSON200.Results {
            fmt.Printf("Strike: %.2f  Exp: %s  IV: %.4f\n",
                opt.Details.StrikePrice, opt.Details.ExpirationDate, *opt.ImpliedVolatility)
            if opt.Greeks != nil {
                fmt.Printf("  Delta: %.4f  Gamma: %.4f  Theta: %.4f  Vega: %.4f\n",
                    opt.Greeks.Delta, opt.Greeks.Gamma, opt.Greeks.Theta, opt.Greeks.Vega)
            }
        }
    }
}
```

Go uses pointer params: `rest.Ptr(value)`. Greeks is a pointer (may be nil). Response data is in `resp.JSON200`.
