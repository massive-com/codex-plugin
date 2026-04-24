# Kotlin / JVM scaffold

Dependencies: `build.gradle.kts`

```kotlin
plugins {
    kotlin("jvm") version "2.1.10"
    application
}

java {
    sourceCompatibility = JavaVersion.VERSION_21
    targetCompatibility = JavaVersion.VERSION_21
}

tasks.withType<org.jetbrains.kotlin.gradle.tasks.KotlinCompile> {
    compilerOptions {
        jvmTarget.set(org.jetbrains.kotlin.gradle.dsl.JvmTarget.JVM_21)
    }
}

application {
    mainClass.set("MainKt")
}

repositories {
    maven("https://jitpack.io")
    mavenCentral()
}

dependencies {
    implementation("com.github.massive-com:client-jvm:v5.1.2")
    implementation("io.github.cdimascio:dotenv-kotlin:6.4.1")
}
```

Also create `settings.gradle.kts`:

```kotlin
rootProject.name = "$0"
```

Entry point: `src/main/kotlin/Main.kt`

Run command: `./gradlew run`

Generate a Gradle wrapper so users don't need Gradle pre-installed:

```bash
# Run this after creating the project (requires Gradle installed on the dev machine)
cd $0 && gradle wrapper
```

If Gradle is not available, include a note in the README that the user needs to install Gradle or the Gradle wrapper first.

`.gitignore` additions: `.gradle/`, `build/`

## rest Main.kt

```kotlin
import io.github.cdimascio.dotenv.dotenv
import io.polygon.kotlin.sdk.rest.PolygonRestClient
import io.polygon.kotlin.sdk.rest.AggregatesParameters
import java.time.Instant
import java.time.ZoneOffset

fun main() {
    val env = dotenv()
    val apiKey = env["MASSIVE_API_KEY"]

    val client = PolygonRestClient(apiKey)

    val params = AggregatesParameters(
        ticker = "AAPL",
        multiplier = 1,
        timespan = "day",
        fromDate = "2025-01-01",
        toDate = "2025-06-01",
        unadjusted = false,
        limit = 30,
        sort = "asc"
    )

    val result = client.getAggregatesBlocking(params)

    println("Status: ${result.status}")
    println("Results count: ${result.resultsCount}")
    println()

    result.results?.forEach { bar ->
        val ts = bar.timestampMillis
        val dt = if (ts != null) Instant.ofEpochMilli(ts).atZone(ZoneOffset.UTC).toLocalDate() else "N/A"
        println("$dt  O:${bar.open}  H:${bar.high}  L:${bar.low}  C:${bar.close}  V:${bar.volume}")
    }
}
```

The SDK package is `io.polygon.kotlin.sdk`. Bar fields use full names: `open`, `high`, `low`, `close`, `volume`, `timestampMillis` (Unix ms, nullable Long). Results are nullable (`results?`).

Auth: pass the API key directly to the `PolygonRestClient` constructor. Use `dotenv-kotlin` for .env loading.

Pagination: manual cursor-based via `nextUrl` on the response. No auto-pagination.

## websocket Main.kt

```kotlin
import io.github.cdimascio.dotenv.dotenv
import io.polygon.kotlin.sdk.websocket.*

fun main() {
    val env = dotenv()

    val client = PolygonWebSocketClient(
        apiKey = env["MASSIVE_API_KEY"],
        feed = Feed.Delayed,
        market = Market.Stocks,
        listener = object : DefaultPolygonWebSocketListener() {
            override fun onAuthenticated(client: PolygonWebSocketClient) {
                println("[status] authenticated, subscribing to T.AAPL")
                client.subscribeBlocking(listOf(
                    PolygonWebSocketSubscription(PolygonWebSocketChannel.Stocks.Trades, "AAPL")
                ))
            }

            override fun onDisconnect(client: PolygonWebSocketClient) {
                println("[status] disconnected")
            }

            override fun onError(client: PolygonWebSocketClient, error: Throwable) {
                println("[error] ${error.message}")
            }

            override fun onReceive(client: PolygonWebSocketClient, message: PolygonWebSocketMessage) {
                when (message) {
                    is PolygonWebSocketMessage.StocksMessage.Trade ->
                        println("Trade: ${message.ticker} @ ${message.price}")
                    is PolygonWebSocketMessage.StatusMessage ->
                        println("[status] ${message.status}: ${message.message}")
                    else -> println(message)
                }
            }
        }
    )
    client.connectBlocking()
}
```

WebSocket package: `io.polygon.kotlin.sdk.websocket`. Client uses listener pattern with `onAuthenticated`, `onReceive`, `onDisconnect`, `onError` callbacks. Subscribe after authentication. Always implement all four listeners plus a `StatusMessage` branch in `onReceive` — without them, auth failures and plan-tier rejections look like silent hangs.

For **websocket type** in Kotlin, add to `build.gradle.kts`:

```kotlin
implementation("com.github.massive-com:client-jvm:v5.1.2")
```

The WebSocket client is included in the same artifact.
