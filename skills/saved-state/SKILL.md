---
name: saved-state
description: What survives process death in a KMP Compose app and what doesn't — the MVIViewModel's state (memory only), rememberSaveable for UI state, serializable routes, the local database and settings for what the person would miss, drafts for long forms, and platform types kept out of common code. Use when adding or reviewing a ViewModel, a route, a form, or anything received from an intent or deep link.
---

# Saved state

| Holder | Rotation | Process death |
|---|---|---|
| The ViewModel's state (`com.shomero.kmp:mvi`), `@Single`, repository memory | ✅ | ❌ |
| `remember` | ❌ | ❌ |
| `rememberSaveable`, the back stack of `@Serializable` routes | ✅ | ✅ |
| Local database, settings `Store` | ✅ | ✅ |

`MVIViewModel` keeps its state in memory: it survives rotation and lives as long as the screen's entry is on the back stack, but isn't restored after the OS kills the app. The library doesn't cover process death yet.

Rule: if losing it after the OS kills the app would annoy the person, put it somewhere that survives.

## Where each thing goes

- **Pure UI state** (expanded, selected tab, scroll, a dialog open) → `rememberSaveable` / `rememberLazyListState` in the screen. A custom type gets a `Saver`. Scroll state comes from `rememberLazyListState()`, `rememberLazyGridState()` or `rememberScrollState()`, never `remember { LazyListState() }`, which is not saved.
- **Never lock the orientation or add `android:configChanges`** to stop a screen losing its state: that hides the bug instead of fixing it.
- **Where the person is and what they opened** → the route's fields (`Route.Note(id)`): Navigation 3 restores the back stack only if every key is `@Serializable` with serializable fields. No `Uri`, platform file or lambda in a key; pass an id or a string.
- **Fetched data** → the local database, re-observed by `load()` after a restart.
- **What the person typed in a long form** → a draft in a local data source (database or `Store`), saved as they type and cleared on submit. A short form can lose it.
- **Received from outside** (share intent, deep link, camera result) → turn it into a route or a draft at once, as a string; never hold it only in a `MutableStateFlow`.

## Platform types in common code

```kotlin
// commonMain
interface SharedFiles { fun restore(saved: String): SharedFile?; fun save(file: SharedFile): String? }
// androidMain: @Single(binds = [SharedFiles::class]) class DefaultSharedFiles : SharedFiles   (Uri.parse / uri.toString())
// other targets: none; callers take SharedFiles?
```

An interface each platform implements, never `if (isAndroid)`.

## Checklist

- [ ] Typed input in long forms survives `adb shell am kill <package>` (app backgrounded, then reopened from recents)
- [ ] Every route is `@Serializable` with serializable fields
- [ ] No outside-received value lives only in a `MutableStateFlow`
- [ ] `remember` holding UI state the person would miss → `rememberSaveable`

Enable "Don't keep activities" in developer options to test continuously.
