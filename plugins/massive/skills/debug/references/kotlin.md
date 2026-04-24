# Kotlin-specific Massive errors

## `NullPointerException` or missing API key

- Pass the API key directly to the `PolygonRestClient` constructor: `PolygonRestClient(apiKey)`.
- Use `dotenv-kotlin` to load `.env`: `val env = dotenv(); val client = PolygonRestClient(env["MASSIVE_API_KEY"])`.

## Null results

- `results` is nullable. Always use safe calls: `result.results?.forEach { ... }`.
- Greeks is nullable: `opt.greeks?.let { g -> ... }`.

## Wrong dependency coordinates

- The SDK is on JitPack (NOT Maven Central). Gradle dep: `implementation("com.github.massive-com:client-jvm:v5.1.2")`.
- Must add `maven("https://jitpack.io")` to repositories.

## Wrong class names or imports

- The SDK package is `io.polygon.kotlin.sdk`, not `org.openapitools` or `io.massive`.
- REST client: `io.polygon.kotlin.sdk.rest.PolygonRestClient`, not `DefaultApi`.
- Methods use `Blocking` suffix: `getAggregatesBlocking(AggregatesParameters(...))`, not `getStocksAggregates(...)`.
- Bar fields use full names: `bar.open`, `bar.high`, `bar.close`, `bar.volume`, `bar.timestampMillis` (not `bar.o`, `bar.h`, etc.).

## WebSocket

- Import from `io.polygon.kotlin.sdk.websocket.*`.
- Classes use `Polygon` prefix: `PolygonWebSocketClient`, `PolygonWebSocketChannel`, `PolygonWebSocketMessage`.
- Subscribe after `onAuthenticated` callback fires.
- Three connect variants: `connect()` (suspend), `connectBlocking()`, `connectAsync(callback)`.

## Rate-limit retry

Wrap SDK calls with an exponential-backoff helper:

```kotlin
fun <T> withBackoff(maxAttempts: Int = 5, fn: () -> T): T {
    repeat(maxAttempts) { attempt ->
        try {
            return fn()
        } catch (e: Exception) {
            val isRateLimit = e.message?.contains("429") == true
            if (!isRateLimit || attempt == maxAttempts - 1) throw e
            Thread.sleep((1L shl attempt) * 1000)  // 1s, 2s, 4s, 8s, 16s
        }
    }
    error("unreachable")
}

val result = withBackoff { client.getAggregatesBlocking(params) }
```
