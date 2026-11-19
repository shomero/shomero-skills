---
name: feature-monitoring
description: Monitor a KMP app in production — crashes, ANRs, slow screens and DI timings reported per app and environment, consent before telemetry, screen names from nav key class names, analytics and logs as breadcrumbs, release mappings uploaded, and ANRs treated as bugs. Use when adding crash reporting or performance monitoring, a new environment, or investigating a production issue.
---

# Feature: monitoring

## What to report

| Signal | From |
|---|---|
| Crashes, ANRs | the monitoring SDK in each app shell |
| Screen visits and timing | a screen tracker keyed on the current nav key |
| DI graph timings | the Koin monitor (see `kotzilla`) |
| Breadcrumbs | analytics event names + `w`/`e` logs through a log sink |

## Rules

- **One monitoring project per app × environment**: staging noise never pollutes prod numbers.
- Keys per environment in gitignored files, written by CI. The build fails loudly when they're missing.
- **Consent first** where the project requires it: buffer until granted, `forgetMe()` on revoke.
- **Screen name = the nav key's class name**, never its arguments (keys can carry ids or worse). Wrap the tracker in `key(navKey)` so a tab switch is a new visit.
- Breadcrumbs carry **names only**, never params.
- **Upload R8 mappings** for every release from the module that runs R8, tagged with the build id; keep nav key names (see `r8-release`).
- Same user id as analytics, set on login, cleared on logout.

## Startup cost

Monitoring itself must not slow the start: initialise it from the DI start (`monitoring()` last in `startKoin`), not an extra startup initializer.

## ANRs and jank are bugs

- An ANR seen anywhere (cold start included) gets a ticket with `adb shell dumpsys activity anr` and the logcat around it.
- Common causes: SDK clients built on Main, kotlin-reflect booting on Main (an embedded server engine), DI graph resolution in a splash condition. Fix: lazy + background dispatcher (see `feature-startup`).
- Per-frame recomposition: read animated values in the draw phase (`drawBehind`, `graphicsLayer`), not composition.

## Investigating

Start from the version and screen, then the issue's dependency tree and timings, fix, bump the version, capture a fresh local session to verify. A fix without a new session is reported as unverified.
