---
name: feature-auth-soft-gate
description: Build an app people can use signed out (a soft gate) — public screens for guests, routes that declare requiresLogin, navigate() that detours through sign-in and returns, actions that need an account replayed after login, data that swaps between guest and user, and a logout that keeps the public cache and only prunes protected screens. Use for consumer, content, marketplace or community apps with something to see before signing up.
---

# Feature: soft-gated app

Guests browse; sign-in is asked for only when they do something that needs an account. Pick this when there's public value before sign-up (see `feature-auth` to choose).

## Per-key gate

```kotlin
@Serializable data object Feed : Route                                   // public
@Serializable data class NoteDetail(val id: NoteId) : Route              // public
@Serializable data object Library : Route { override val requiresLogin = true }
```

`requiresLogin` defaults to `false`; protected keys say so. The `Navigator` does the rest (see `navigation3`):

| Call | Signed out |
|---|---|
| `navigate(Library)` | pushes sign-in over the current screen, remembers `Library` |
| `onLoginSuccess { target -> below + listOfNotNull(target) }` | replaces sign-in with `Library`, over what was below (`below = navigator.toList().dropLast(1)`) |
| `reset(navigator.toList().filterNot { it.requiresLogin })` | logout: drops every protected route; the user stays on the public screen. Not `onLogout()`, which leaves only sign-in |

Login is **pushed over** the current screen (or shown as a sheet), never a stack reset: back from login returns to what the guest was doing.

## Actions that need an account

```kotlin
is NotesUiEvent.Save -> if (isSignedIn) save(event.id)
    else { pendingActions.set(PendingAction.Save(event.id)); update { copy(action = NotesUiAction.SignIn) } }
// in load(), merged: once the observed user is non-null, replay and clear what's pending
session.userId().filterNotNull().onEach { pendingActions.take()?.let(::replay) }
```

- The pending action lives in a small local data source (see `feature-settings`), not only in the ViewModel: sign-in replaces the stack, and the app can be killed during it.
- The prompt says why: "Sign in to save this note", not a bare login wall.
- Cancelling sign-in drops the pending action silently.

## Data that depends on who's looking

- **Swap on identity**: `observeUserId().flatMapLatest { id -> if (id == null) guestFeed() else userFeed(id) }`. Every user-specific stream follows the same shape, so signing in updates open screens.
- **Split the cache**: public tables (catalog, public content) vs user tables (library, settings, drafts). Logout wipes only user tables; the guest keeps a fast public app.
- **Guest-made data** (a cart, local favorites, onboarding answers): decide once, per entity, **merge into the account on sign-in or discard**, and record the decision in the project's CLAUDE.md. Never silently keep it on the device for the next user.
- Analytics: anonymous id while a guest, `identify(userId)` on sign-in, `reset()` on logout (see `feature-analytics`).

## Rules

- A protected screen pushed onto the back stack directly instead of with `navigate()` is an auth bypass.
- Deep links to a protected key go through sign-in, then on; to a public key, straight there.
- Session loss mid-use: drop the protected routes (as on logout) and show a sign-in prompt on the public screen, not a crash.
- Tabs mixing public and protected (a "Library" tab): the tab is visible, tapping it signed out goes through `navigate()`.

## Checklist

- [ ] Every route that shows user data declares `requiresLogin = true`
- [ ] Each "needs an account" action has a pending replay and a reason in the prompt
- [ ] Logout keeps public cache, wipes user tables, removes protected screens
- [ ] Guest data merge-or-discard decided per entity
- [ ] Tests: redirect and return, pending replay after login, cancel drops it, logout prunes the stack
