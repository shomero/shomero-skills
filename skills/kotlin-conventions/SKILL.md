---
name: kotlin-conventions
description: Kotlin and KMP coding rules for commonMain — Rule of Three, no object or expect fun for behaviour, imports over fully-qualified names, errors as exceptions and never runCatching around a suspend call, value classes for ids, coroutines and dispatchers on native, kotlinx.datetime with explicit time zones, and the APIs that don't exist in common code. Use when writing or reviewing any Kotlin in a KMP project.
---

# Kotlin conventions

## Structure

- **Rule of Three.** Two copies are a coincidence; the third is the signal. Before writing the third, lift the existing two into a shared module (`git mv` the original so history follows), delete the duplicates, and repoint every caller in the same change.
- **No `object` for behaviour.** An interface with a `Default<Interface>` bound in Koin, so tests and platforms can swap it. Pure functions and constants are top-level declarations in a file named after the group, not members of an `object`, each named to read on its own at the call site: `routePath(route)`, not `RouteUrls.path(route)`; `StandardEasing`, not `Motion.Standard`.
- **Import, never inline a fully-qualified name.** Use `import … as …` on a clash.
- **Small, named types over comments** (see `code-comments`).
- **Ids are value classes**: `@JvmInline @Serializable value class NoteId(val value: String)`. Two ids can't be swapped by accident.
- **Sealed interfaces** for closed sets of events, actions and states; `when` without `else` so a new case fails the build.

## Errors

- **Errors are exceptions, not `Result<T>`.** The API client throws `OfflineException` (no answer) or `ReasonException(ErrorReason)`; data sources, repositories and use cases return values or `Flow`s and let them propagate. The ViewModel is where they stop: `catch` on the flow, `failed(e)` into a `ScreenState` (see `mvi-viewmodel`).
- **Never `runCatching` around a suspend call**: it catches `CancellationException`, and a cancelled coroutine keeps running. On a flow, use `catch`, which never sees the cancellation. In a coroutine, catch the exceptions you expect, or rethrow `CancellationException` first.
- Never surface `exception.message` to users or analytics.

## Coroutines

- **No `GlobalScope`.** Use `viewModelScope`, or the app's injected `@Single` scope.
- **Inject `CoroutineDispatcher`**; never hard-code `Dispatchers.Default` (tests can't control it).
- **`Dispatchers.IO` doesn't exist on Kotlin/Native.** In `commonMain` use the injected dispatcher; `IO` is fine in `androidMain`/`jvmMain`.
- **`Dispatchers.Main` in `commonMain`** needs `kotlinx-coroutines-swing` on desktop.
- A `Job` you keep must be cancelled in `onCleared()` or when replaced.
- Flow-returning functions return the flow; the caller collects.

## Not in commonMain

`java.*`, `android.*`, `java.time`, `String.format`, `Locale`, `System.currentTimeMillis`, `java.io.File`. Use:

| Instead of | Use |
|---|---|
| `System.currentTimeMillis()` | `Clock.System.now()` (`kotlin.time`) |
| `LocalDate.now()` | `Clock.System.todayIn(TimeZone.currentSystemDefault())` |
| `String.format` | a small `expect`/`actual` formatter, or string templates |

**Always pass the `TimeZone` explicitly.** Day-boundary logic that assumes UTC is wrong in every other zone.

## expect/actual

- Only for real platform APIs, and keep the actual thin. **No `expect object` or `expect fun` handing out a service**: an interface in common, and each platform source set annotates its own implementation for Koin. A platform without one provides none, and the caller takes it as nullable.
- A new `expect` member needs an `actual` in **every** target the module builds. Check the module's targets first.
- A custom source set with `dependsOn` needs `applyDefaultHierarchyTemplate()`, or `iosArm64Main → iosMain` is cut and actuals silently vanish.
