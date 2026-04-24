# Kotlin SDK pattern for options chain

```kotlin
import io.github.cdimascio.dotenv.dotenv
import io.polygon.kotlin.sdk.rest.PolygonRestClient
import io.polygon.kotlin.sdk.rest.SnapshotOptionsChainParameters

fun main() {
    val env = dotenv()
    val client = PolygonRestClient(env["MASSIVE_API_KEY"])

    // Get current price
    val lastTrade = client.getLastTradeBlockingV2("SPY")
    val spot = lastTrade.results?.price

    // Fetch options chain
    val params = SnapshotOptionsChainParameters(
        underlyingAsset = "SPY",
        strikePriceGte = 500.0,
        strikePriceLte = 600.0,
        expirationDateGte = "2025-06-01",
        expirationDateLte = "2025-06-30",
        contractType = "call",
        order = "asc",
        limit = 250
    )
    val chain = client.getSnapshotOptionsChainBlocking(params)

    chain.results?.forEach { opt ->
        println("Strike: ${opt.details?.strikePrice}  Exp: ${opt.details?.expirationDate}")
        println("  IV: ${opt.impliedVolatility}  OI: ${opt.openInterest}")
        opt.greeks?.let { g ->
            println("  Delta: ${g.delta}  Gamma: ${g.gamma}  Theta: ${g.theta}  Vega: ${g.vega}")
        }
    }
}
```

SDK package is `io.polygon.kotlin.sdk`. Auth: pass API key to `PolygonRestClient` constructor. Gradle dep: `com.github.massive-com:client-jvm:v5.1.2` from JitPack.
