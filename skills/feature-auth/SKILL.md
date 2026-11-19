---
name: feature-auth
description: The sign-in core every KMP app shares, and how to choose between a hard gate (everything behind sign-in) and a soft gate (guests can browse) — auth state, the auth flow module, session restored at startup, encrypted session storage per platform, social sign-in behind an interface each platform implements, and a logout that wipes data even when the server call fails. Use when adding or changing login, sign-up, social sign-in, session handling or logout, then follow feature-auth-hard-gate or feature-auth-soft-gate.
---

# Feature: auth

## Choose the gate first

| | Hard gate | Soft gate |
|---|---|---|
| Signed out sees | only login / sign-up | public screens |
| Fits | B2B, internal, health, finance, anything with no public content | consumer, content, marketplace, community |
| Gate lives in | one root effect that resets the whole stack | each route's `requiresLogin` + the `Navigator` |
| Login success | auth state flips, root resets the stack | `onLoginSuccess`, back to the remembered screen |
| Guest data | none | public cache + merge-or-discard per entity |
| Logout | wipe everything, back to login | wipe user data, prune protected screens, stay |
| Skill | `feature-auth-hard-gate` | `feature-auth-soft-gate` |

Write the choice in the project's CLAUDE.md; mixing both in one app produces screens that are half-protected.

## Shared pieces

| Piece | Where | Does |
|---|---|---|
| `Route.Auth` (sealed, inside the app's `Route`) | `apps/<app>/composeApp` | sign-in, sign-up, reset: never `requiresLogin` |
| `SignInScreen(onSignedIn, onSignUp)`, … | `features/auth` | the flow's screens; the app maps each `Route.Auth` case in `App.kt` |
| `ObserveAuthStateUseCase` | `features/auth/domain` | `Unknown / SignedIn(userId) / SignedOut` |
| `AuthSessionInitializer` | `services/auth` | restores the session at startup (see `feature-startup`) |
| `LogoutUseCase` | `services/auth` | the ordered logout below |
| `LocalDataWiper` | `services/auth` | clears the tables the gate says to clear |

## Session

- Restored by an `Initializer` (see `feature-startup`) that **never blocks the app**: it throws or returns a failure instead of hanging, under its `timeout`. Worst case the user starts signed out.
- A stored session for a user the server no longer knows: load the user with a timeout, sign out if it never arrives.
- Storage: Android `EncryptedSharedPreferences` (AES256_GCM), iOS Keychain; desktop and web say plainly when unencrypted. Never tokens in plain settings. `android:allowBackup="false"`.
- One token refresh at a time, shared by every client (see `feature-network`).

## Social sign-in

A `GoogleSignIn` interface with a sealed `Success / Cancelled / Error / NotAvailable` result, implemented by each platform source set that has it; the caller takes it as nullable. Cancel shows nothing; the button is disabled until configured. Details: `google-sign-in`.

## Logout, in order

1. Confirm or discard in-progress work (an upload, an unsaved form).
2. Sign out of dependent services that keep their own identity (purchases, analytics `reset()`).
3. Remote sign-out, then **wipe local data whatever the result** (everything for a hard gate, user tables for a soft gate).
4. Clear per-user settings and consent state.
5. The gate takes over the navigation.

## Checklist

- [ ] The gate type is written in CLAUDE.md
- [ ] Prefilled staging credentials are blank in release builds
- [ ] Every new cached table is classified (wiped on logout or public) and in the wipe if needed
- [ ] Tests: login success and failure, social cancel, logout wipes even when the remote call fails
