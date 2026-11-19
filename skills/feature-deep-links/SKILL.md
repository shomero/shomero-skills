---
name: feature-deep-links
description: Handle deep links and share links in a KMP app — every platform entry point offers the URL to one DeeplinkBridge, a resolver maps it to routes or "await login", the stack is rebuilt with parents, and routes map back to share links. Use when adding a link, a share button, App Links / Universal Links, or a notification that opens a screen.
---

# Deep links

## Shape

```kotlin
@Single class DeeplinkBridge { private val pending = MutableStateFlow<String?>(null)
    val link: StateFlow<String?> = pending.asStateFlow()
    fun offer(url: String) { pending.value = url }; fun consume() { pending.value = null } }
```

Entry points only **offer**:

| Platform | Where |
|---|---|
| Android | `MainActivity.onCreate` **and** `onNewIntent`: `intent.dataString?.let(bridge::offer)` |
| iOS | SwiftUI `.onOpenURL` and `.onContinueUserActivity` → a Kotlin `offer(url:)` |
| Desktop | OS URL handler → `offer` |
| Web | the address bar after `#` (see `navigation3`) |

## Resolve, then apply

A `DeeplinkEffect` at the root resolves the pending link to one of:

- `Apply(routes)` → `navigator.reset(routes)`: the parents, then the target (`Home`, `Notes`, `Note(id)`), so back works. The web's `routeStack(path)` (see `navigation3`) is the same mapping.
- `AwaitLogin` → keep it pending; apply after sign-in.
- `Dismiss` → unknown or invalid link; consume and ignore.

Resolution is a tested use case: every route, a malformed id, a signed-out user.

## Share links

`routePath(route)` maps a route back to its address; the link is the app's domain plus it, shared through a `ShareSheet` interface each platform implements. Ids only in the link.

## Android manifest

- `https` links: `android:autoVerify="true"` plus the `assetlinks.json` on the domain.
- Validate every id from a link before loading; a link is untrusted input.
- Only the launcher activity is exported.

## Notifications

A notification tap is a deep link: put the URL in the notification's intent and let the same bridge handle it. In-app notification lists resolve each item to a `NotificationTarget` enum, mapped to navigation by the app.
