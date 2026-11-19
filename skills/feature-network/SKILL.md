---
name: feature-network
description: Set up the network layer of a KMP app — one API client owned by the app (Ktor HttpClient or a gRPC channel) that turns every failure into OfflineException (no answer) or ReasonException(ErrorReason), auth headers with a single-flight token refresh, retries that stay bounded, JSON settings that survive real servers, and how failures reach the screen through ScreenState and retryWhenOnline. Use when adding a client, an endpoint group, auth headers, retries, or error handling.
---

# Network client

## One client, provided by the app

- The app's DI root builds the single `HttpClient` (or gRPC `Channel`) from build config and binds it as `@Single`. Several clients → qualifiers (`@Named("authenticated")`).
- `expectSuccess = true`, so a non-2xx answer throws. Install `ContentNegotiation(json)` and `Logging` routed to the app logger, with **headers and bodies off in release** (tokens, user data).
- Never build an `HttpClient` per call. Close the clients you own. Tests pass a `MockEngine` client instead.

```kotlin
val json = Json { ignoreUnknownKeys = true; coerceInputValues = true; explicitNulls = false }
```

A default value covers an **absent** key, not an explicit `null`. Without `coerceInputValues`, one `null` row fails the whole list.

## Failures, mapped once

```kotlin
class ApiClient(val http: HttpClient) {
    suspend fun <T> request(block: suspend HttpClient.() -> T): T = try {
        http.block()
    } catch (e: CancellationException) { throw e
    } catch (e: ResponseException) { throw ReasonException(e.reason(), cause = e)   // 404 NotFound, 401/403 NotAllowed, 5xx Server
    } catch (e: SerializationException) { throw ReasonException(ErrorReason.Server, cause = e)
    } catch (e: Exception) { throw OfflineException(e) }                            // no answer: no network, timeout, unknown host
}
```

- gRPC: `UNAVAILABLE` and `DEADLINE_EXCEEDED` → `OfflineException`; `NOT_FOUND`, `PERMISSION_DENIED`/`UNAUTHENTICATED` and the rest → `ReasonException` with their `ErrorReason`.
- Log the status and the path, never a body, a token or a header. Never decide by an exception's message.

## Up the stack

```
API client:   OfflineException or ReasonException(ErrorReason), nothing else
data source:  calls through request { }, catches nothing
repository:   returns values or Flows; lets them propagate, never .catch { emit(null) }
ViewModel:    load() … .retryWhenOnline(connectivity) { failed(e) } .catch { failed(e) } into a ScreenState
UI:           ScreenStateContent: offline and error with Retry, a string per ErrorReason
```

- Never show `exception.message`: it can carry the raw server body.
- **Offline is a state**, not a toast on every call: content already on screen stays under a banner, and `retryWhenOnline` reloads once the network is back (1, 2, 4, 8, 16, then every 30 s).
- A failed action (save, delete) shows its `ErrorReason`'s string; the typed input stays.

## Auth

- The client adds the auth header per request and refreshes **once** on 401, then retries once. A refresh that fails is `NotAllowed`: sign out.
- **Single-flight refresh**: guard it with a `Mutex`, so ten concurrent 401s cause one refresh.
- Credentials go in the request **body**, never in query parameters (URLs end up in logs).

## gRPC

- A `BaseApi` owns a lazy, mutex-guarded `Channel` (port 443, keep-alive), interceptors for logging and client metadata (`x-platform`, app version), and builds auth metadata **per attempt** so a retry after refresh carries the new token.
- Each domain API: `class DefaultNotesApi(base: BaseApi) : NotesApi, BaseApi by base`.
- Layers: proto → `Backend*` model in the API module → domain model in the service. Never store proto classes.

## Retry

- Loads that fail offline are retried by the ViewModel's `retryWhenOnline`, so retries stop when the screen leaves. Don't add a second loop in the client.
- The client retries only what is safe: an idempotent call on 429/503 with `Retry-After`, a few times with jitter. **Exhausted retries surface**: throw, so the screen shows a state instead of waiting for nothing.
- Recreate a gRPC channel on `UNAVAILABLE`.

## Lifecycle

If the client offers `pause()` / `resume()` for app background and foreground, wire it to the process lifecycle or delete it. Unwired lifecycle code rots.
