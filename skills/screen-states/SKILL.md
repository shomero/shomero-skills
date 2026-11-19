---
name: screen-states
description: The four states of a screen that loads data — loading, content, error and offline as one ScreenState per part, empty as content, content kept under a banner on a failed reload, an offline state that retries by hand and by itself, errors that never loop, the copy each state shows, how to check them, and the shapes to write when the project has no com.shomero.kmp:mvi. Use when a screen loads data, shows a blank screen or a raw error, or when adding or reviewing its loading, error, offline and empty states.
---

# Screen states

Every screen that loads data is loading, showing content, in error, or offline. Model all four: without them a slow network shows a blank screen, and a lost connection a raw exception.

| Library | Gives you |
|---|---|
| `com.shomero.kmp:mvi` | `ScreenState<T>`: `Loading`, `Content(data, refreshing, failure)`, and the two `Failure`s, `Error(reason)` and `Offline(retrying)`; `ErrorReason`, `ReasonException`, `loadingAgain()`, `failed(e)`, `dataOrNull` |
| `com.shomero.kmp:connectivity` | `Connectivity.online`, `OfflineException`, `Flow.retryWhenOnline(connectivity)`, `FakeConnectivity` |
| `com.shomero.cmp:mvi-states` | `ScreenStateContent`, the `LoadingState` and `FailureState` it draws, `RefreshFailedBanner` for a failure over content |

The ViewModel sets the state from `load()` (see `mvi-viewmodel`), the screen draws it (see `compose-screen`), and the API client decides what is offline and what is an error (see `feature-network`). A project with its own versions uses those.

## Rules

- **One field per part.** A screen with independent parts (a header and a feed) gets one `ScreenState` field each, so a slow part doesn't block the rest. Never `data == null && error == null` to mean "loading".
- **Empty is content**: `Content` with an empty list. The content draws its own empty state, with one line on what to do next.
- **Keep content on a failed reload.** `failed(e)` puts the failure in `Content.failure`, and `RefreshFailedBanner` shows it over the content, worded for offline or error.
- **Offline retries by hand, and by itself.** The offline state always has a Retry button, which calls `reload()` straight away, whatever `online` says; `retryWhenOnline` also reloads when the network comes back. The same call reconnects a live watch that dropped.
- **`online` only triggers a retry.** Only a request that failed as offline shows the offline state, never `online == false` alone. When the device says it's online but requests still fail (a captive portal, the server down), the backoff runs and Retry stays on screen.
- **Errors don't loop.** An error isn't retried automatically: show it, with Retry. "Not found" and "not allowed" also offer a way out (back, or home), since retrying won't fix them.
- **One exhaustive draw**: `ScreenStateContent`, or one `when` without `else`, so a new state fails the build instead of drawing nothing. Loading, error and offline fill the space the content would, inside the screen's frame, so the header, navigation and Back stay usable.
- **Loading shows after about 300 ms** (a skeleton or a progress indicator), so a fast load doesn't flash.
- **Copy.** Offline says the device is offline, not that something broke, and that the screen reloads once it's back. Its Retry button shows progress while `Offline.retrying` is true. The exception is logged with its stack trace; the person sees a string per `ErrorReason`.

## Check

- **Tests** (see `testing-kmp`), plus: nothing reloads after the screen has left.
- **Previews**: one for each state.
- **By hand**: go offline (browser network throttling set to Offline, airplane mode on a phone), open the screen, see the offline state and Retry. Go back online and watch the screen reload by itself.

## Without the libraries

Write the same shapes in `libraries/`, with the same names, so the project can move to the libraries later without changing its screens:

```kotlin
sealed interface ScreenState<out T> {
    data object Loading : ScreenState<Nothing>
    /** [failure] is why the last reload failed, shown as a banner over the data that stays on screen. */
    data class Content<T>(val data: T, val refreshing: Boolean = false, val failure: Failure? = null) : ScreenState<T>
    sealed interface Failure : ScreenState<Nothing>
    data class Error(val reason: ErrorReason) : Failure
    /** [retrying] is true while it tries again, so the Retry button can show progress. */
    data class Offline(val retrying: Boolean = false) : Failure
}
enum class ErrorReason { NotFound, NotAllowed, Server, Unknown }
open class ReasonException(val reason: ErrorReason, message: String? = null, cause: Throwable? = null) : Exception(message, cause)

fun <T> ScreenState<T>.loadingAgain(): ScreenState<T> = when (this) {
    is ScreenState.Content -> copy(refreshing = true)
    is ScreenState.Offline -> copy(retrying = true)
    else -> ScreenState.Loading
}

interface Connectivity {
    /** True while the device has a network. A hint for when to retry, not proof the server can be reached. */
    val online: StateFlow<Boolean>
    suspend fun awaitOnline() { online.first { it } }
}
class OfflineException(cause: Throwable? = null) : Exception("The device couldn't reach the server", cause)
```

- `failed(e)` rethrows cancellation and logs anything that isn't offline. `OfflineException` becomes `Offline`, `ReasonException` becomes `Error(reason)`, anything else `Error(Unknown)`; over content, it keeps the content with `failure` set.
- `retryWhenOnline(connectivity) { e -> }` is `retryWhen` on `OfflineException`: report it, wait for `awaitOnline()`, back off (1, 2, 4, 8, 16, then 30 seconds, which also lets the network settle), try again. It resets the backoff once something loads.
- `Connectivity` per platform: Android `ConnectivityManager.registerDefaultNetworkCallback`, counting `NET_CAPABILITY_VALIDATED` as online; iOS `NWPathMonitor`; web the window's `online` and `offline` events; desktop a poll of `NetworkInterface` every few seconds.
- Run the platform listener only while someone collects it (`callbackFlow` + `shareIn(WhileSubscribed)`). `awaitOnline()` waits for the listener's own answer, not a stale value.
