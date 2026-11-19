---
name: kotzilla
description: Set up Kotzilla monitoring for Koin in a KMP app — one Kotzilla app per app×environment, per-env kotzilla-<env>.json kept out of git, monitoring() last in startKoin, consent-gated telemetry, screens from nav keys via KotzillaTrackScreen, analytics events as timeline marks, and R8 mapping upload. Use when adding Kotzilla, a new app or environment, screen tracking, or reading its issues through the Kotzilla MCP.
---

# Kotzilla

## Setup

- Apply the Kotzilla Gradle plugin to each app's `composeApp`; `monitoring()` is the **last** call inside `startKoin`.
- **One Kotzilla app per app × environment** (`App`, `App Stg`), so staging noise never lands in prod.
- Keys live in `composeApp/kotzilla-<stg|prod>.json`, **gitignored**. CI writes them from env vars holding the whole file base64-encoded. The plugin fails the build when the file for the selected env is missing: copy them into a fresh worktree.
- Pick the file with the same env rule as BuildKonfig (`-Penv` / `Prod` in the task name). `composeApp` has no flavors, so without it a prod build reports into staging.
- Never retype `keyId` / `apiKey`; write the JSON the MCP's `create_app` returns.

## Privacy

- `consentRequired = true`: telemetry buffers until the analytics consent gate grants it (`onConsentChanged`). Zero sessions before anyone accepts is correct.
- `composeInstrumentation = false`, and name screens yourself:

```kotlin
key(navKey) { KotzillaTrackScreen(navKey.analyticsName()) }   // class name only, never arguments
```

`key(navKey)` makes a tab switch a new screen visit instead of renaming the current one.

## Analytics timeline

A `KotzillaAnalyticsConsumer` in the analytics fan-out puts every event on the session timeline as a mark: event **names only**, never params. `setUserId` forwards the same id other consumers get. Revoke → `setConsent(NOT_GRANTED)` + `forgetMe()`.

## Release builds

Apply the plugin to `androidApp` too: R8's `mapping.txt` is produced there, and `assemble`/`bundle` of a release runs `uploadMappingFile<Variant>` with the key inherited from `composeApp`. Set `androidEarlyStart = false` so the app module adds no startup initializer. Without mappings, release data shows `xc`-style names. Keep nav key names (see `r8-release`).

## Using the MCP

`list_apps` → `get_platform_activity` (pick app + version) → `get_screen_performance` → `get_issues` → `get_issue_context` → `get_fix_guidance`. Verify a fix with a fresh local session on the bumped version; an unverified fix is reported as unverified.
