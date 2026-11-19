---
name: feature-logging
description: Logging in a KMP app on com.shomero.kmp:logger — Logger.d/w/e(tag, …) from common code, Logger.minLevel set from the build in initKoin, LoggerSinks.add for crash reporters and dev tools, the network client's logs routed through it, and the rule that tokens, emails, names and bodies never reach a log. Use when adding logs, a log sink, network logging, or reviewing what gets logged.
---

# Feature: logging

`com.shomero.kmp:logger` logs to Logcat, `NSLog`, standard output or the browser console, and hands every line to its sinks on every platform. Don't write your own `expect` logger or sink registry.

## Log

```kotlin
private const val TAG = "Notes"

Logger.d(TAG, "Loaded ${notes.size} notes")
Logger.d(TAG) { "Ids: ${notes.joinToString { it.id.value }}" }   // built only when debug logs are on
Logger.w(TAG, "Slow sync, retrying")
Logger.e(TAG, "Sync failed", error)
```

## Levels, once in initKoin

```kotlin
Logger.minLevel = if (debug) LogLevel.Debug else LogLevel.Warn
```

`debug` comes from the entry point's build config. Release drops `d`; `w` and `e` stay, they feed crash breadcrumbs.

## Sinks

```kotlin
LoggerSinks.add { level, tag, message, throwable ->
    crashReporter.breadcrumb("$level [$tag] $message")
}
```

Sinks: crash-report breadcrumbs, a file sink for support builds; probe's `logs` reads the app's log in debug builds. `LoggerSinks.remove(sink)` detaches one. The sink list is swapped atomically, so attaching from any thread is safe.

## Rules

- **No sensitive data in any log**: no tokens, emails, names, free text, response bodies. Ids alone are fine; an id next to an identifying value isn't. What counts as sensitive is in the project's CLAUDE.md.
- Tag = feature or class (`"Analytics"`, `"NotesRepo"`), a private constant per file.
- Log **which** silence: "disabled: no token" vs "never wired" look identical from outside.
- Network: the API client logs the status and the path of a failure; route Ktor `Logging` to `Logger`, with `HEADERS`/`BODY` in debug only.
- `e` gets the throwable, not `throwable.message` concatenated into the text.
- Use the lazy `d(tag) { … }` when the message is costly to build.
- Don't log in tight loops (per frame, per audio buffer).
- Not for analytics events or errors the person sees: a log is for whoever is debugging.
