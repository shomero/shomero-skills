---
name: mixpanel
description: Wire Mixpanel into a KMP app as one AnalyticsConsumer — a small MixpanelClient interface implemented per platform source set, a null client when no token is set, no autotrack, explicit data residency, envelope as super properties re-registered after reset, consent mirrored to the SDK opt-out, and wire tests. Use when adding Mixpanel, debugging dropped events, or changing identify/logout/consent behaviour.
---

# Mixpanel

Mixpanel is one consumer behind the app's `AnalyticsReporter` (see `feature-analytics`); nothing else calls the SDK.

## Shape

```kotlin
interface MixpanelClient {
    fun track(name: String, params: Map<String, Any?>); fun identify(userId: String); fun reset()
    fun setOptedOut(optedOut: Boolean); fun registerSuperProperties(p: Map<String, Any?>)
    fun setUserProperties(p: Map<String, Any?>); fun flush()
}
// each platform source set: @Single(binds = [MixpanelClient::class]) class DefaultMixpanelClient(config: MixpanelConfig)
class MixpanelAnalyticsConsumer(private val client: MixpanelClient?) : AnalyticsConsumer
```

All ordering rules live in the common consumer; platform clients only make SDK calls. A platform without the SDK binds none, so the consumer takes `MixpanelClient?`. A blank token gives a wired but inert client, so environments differ only in config. **Log which silence it is** ("disabled: no token" vs "initialising against <url>"): the SDK is silent by default and "never wired" looks the same.

## Setup per platform

- Android: `MixpanelAPI.getInstance(context, token, trackAutomaticEvents = false)`. Autotracked screens and taps send data nobody reviewed.
- `setServerURL(...)` to the project's residency host (EU: `https://api-eu.mixpanel.com`). The SDK defaults to the US host and accepts the data there. Residency is fixed at project creation.
- `setUseIpAddressForGeolocation(false)` unless you need location.
- Desktop: `mixpanel-java` is stateless. Merge super properties into each event yourself, generate and persist an install id for pre-login events, and send from the reporter's background scope.
- iOS: needs the SDK via CocoaPods/SPM or cinterop. Until it's wired, return `null` and log it once.

## Identity, envelope, consent

- Login → `identify(userId)`. Logout → `reset()` (new anonymous id, so the next person on the device isn't merged).
- **`reset()` clears super properties**: re-register the envelope right after, or every later event silently loses it.
- The envelope (app version, platform, flavor) goes in **super properties**, not profile properties: profile values are current-value, so every historical event would report the latest app version.
- Consent revoked → `flush()` then `optOutTracking()`; granted → `optInTracking()`. The reporter's gate can't stop events the SDK makes itself.

## Tests

- Consumer in `commonTest` with a fake client: reset re-registers the envelope, revoke flushes before opting out.
- Wire tests that the real SDK produces the expected payload: Robolectric (`androidHostTest`) on Android, JVM on desktop. Run them in PR CI.
- A live smoke test against a test project, skipped without a token.
