---
name: feature-auth-hard-gate
description: Build an app where everything is behind sign-in (a hard gate) — the splash waits for the session, a root observer swaps the whole back stack between the auth flow and the app, login never navigates itself, there is no guest data, deep links wait for sign-in, and logout wipes everything and returns to login. Use for B2B, internal, health, finance or any app with no public content, or when adding sign-in to such an app.
---

# Feature: hard-gated app

Nothing is visible signed out except the auth flow. Pick this when there is no public content (see `feature-auth` to choose).

## Shape

| Piece | Does |
|---|---|
| `Route.Auth` (sealed, inside the app's `Route`) | sign-in, sign-up, reset password: the only routes a signed-out user can reach |
| every other `Route` | app screens. No per-route `requiresLogin` needed: the whole graph is protected |
| `ObserveAuthStateUseCase` | `Unknown / SignedIn(userId) / SignedOut` |
| `RootViewModel` | `showSplash = startup running ‖ auth == Unknown`; owns `Logout` |

## The gate is one effect at the root

```kotlin
LaunchedEffect(authState) {
    val top = navigator.lastOrNull()
    when (authState) {
        is SignedIn -> if (top == null || top is Route.Auth) navigator.reset(Route.Home)
        SignedOut -> if (top !is Route.Auth) navigator.onLogout()           // leaves only sign-in
        Unknown -> Unit                                   // splash still up
    }
}
```

- **Login success never navigates.** The login ViewModel reports success; the auth state flips; the root swaps the stack. One code path for login, sign-up, social sign-in, token expiry and remote sign-out.
- `reset` and `onLogout` replace the whole stack: back from Home never returns to login, and back from login never shows app screens.
- The splash stays until auth is known, so the user never sees login flash before their session loads.

## Rules

- **No guest data.** Nothing is fetched or cached before sign-in, so there's nothing to merge or leak between users.
- **Deep links wait**: a link opened signed out resolves to `AwaitLogin`, stays pending, and applies after sign-in (see `feature-deep-links`).
- **Session loss anywhere** (refresh fails, account deleted, server revokes) is just `SignedOut`: the same effect takes the user to login.
- **Logout** wipes every table and per-user setting (see `feature-auth`), then the effect resets the stack. In-progress work gets a confirm-or-discard guard first.
- Startup, splash and session loading: `feature-startup`.

## Checklist

- [ ] Every screen outside `Route.Auth` is unreachable signed out, by tab, back, deep link or notification
- [ ] Login, sign-up and social sign-in contain no `navigate` call on success
- [ ] Token expiry mid-session lands on login without a crash or a half-rendered screen
- [ ] Logout then login as another user shows none of the first user's data
- [ ] Tests: the effect for each `(authState, top)` pair; deep link pending across sign-in
