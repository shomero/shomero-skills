---
name: feature-analytics
description: Add analytics to a KMP app — typed events generated from a versioned spec, one AnalyticsReporter fanning out to consumers behind a consent gate, screen views from nav keys, CTA events mapped in the ViewModel, ids only, and a RecordingAnalytics test double. Use when adding an event, a screen, a consumer SDK, or consent handling.
---

# Feature: analytics

## Events are generated, not hand-written

- The spec (events, properties, an envelope every event carries, entity ids) lives in its own repo, pinned by a tag in `libs.versions.toml`.
- A build-logic plugin: `syncAnalyticsSpec` downloads the tag, `generateAnalyticsEvents` writes typed classes into a **committed** `generated/` source dir (`kotlin.srcDir(tasks.named("generateAnalyticsEvents"))`). Ids get value-class wrappers.
- CI regenerates and runs `git diff --exit-code -- libraries/analytics/generated`: a spec change and its code land together.

## Reporter

- One `AnalyticsReporter` with a list of `AnalyticsConsumer`s (logging, product analytics, monitoring).
- **Build the consumer list explicitly** in the app's DI root. Cross-module `List<T>` collection silently dropped a consumer once; log the wired consumer names at init.
- **Consent gate defaults closed**, re-checked inside the launch. Granting replays the envelope and user id set while closed. Revoking sends the farewell event, then closes the gate in the same coroutine.
- Each consumer mirrors consent onto its SDK's own opt-out; revoke also calls the SDK's forget-me.
- A monitoring consumer gets event **names only**, never params.
- Telemetry is fire-and-forget on a dedicated `@Named` scope.

## What gets sent

- **Screen views**: `LaunchedEffect(currentKey) { log(screenViewed(currentKey.analyticsName())) }`, where the name is the key's **class name**, never its arguments. Keep nav key names through R8 (`-keepnames class * implements …NavKey`), or screens arrive as `j83`.
- **CTAs**: each ViewModel has `private fun UiEvent.cta(): Cta?`; `handleEvent` calls `event.cta()?.let(analytics::log)`. Typing events return null.
- **Ids only.** Never names, emails, free text or sensitive fields. The logging consumer prints every param, so a leak shows in logcat first.

## Consent UI

A consent dialog rendered at the root, outside the nav graph; its store in settings; re-prompt after refusals on a backoff schedule; a revoke that stops prompting.

## Testing

`RecordingAnalytics`: a real reporter with a recording consumer, a no-op validator, consent pre-granted, exposing `events`, `names()`, `ctas()`. Assert events per ViewModel test. **An unconfined test dispatcher hides ordering races** (revoke-then-send): test those on a `StandardTestDispatcher`.

Needs `analytics`, `analytics-testing` and the codegen plugin from the example project.
