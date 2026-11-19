---
name: feature-settings
description: User preferences in a KMP app on com.shomero.kmp:settings — one Store made per platform and passed to initKoin, one small local data source per concern over store.entry(...) exposing a StateFlow, an interface with a Default implementation and private keys, what may go in plain settings, theme mode, and clearing per-user settings on logout. Use when adding a preference, a settings screen, theme or language switching, or a "don't show again" flag.
---

# Feature: settings

## The Store, once per platform

```kotlin
SharedPrefsStore(context, name = "<app>")    // Android
UserDefaultsStore()                           // iOS
PrefsStore(node = "com/<app>")                // desktop
LocalStorageStore(keyPrefix = "<app>.")       // web
MemoryStore()                                 // previews and tests
```

Each entry point makes its `Store` and passes it to `initKoin(store, …)`, which binds it with `single { store }`: a value only the entry point knows, so one of the few definitions written in Koin's DSL. No `expect` module, no `Settings` of your own.

## One small source per concern

```kotlin
interface ThemeModeLocalDataSource {
    val mode: StateFlow<ThemeMode>
    fun set(mode: ThemeMode)
}

@Single(binds = [ThemeModeLocalDataSource::class])
class DefaultThemeModeLocalDataSource(store: Store) : ThemeModeLocalDataSource {
    private val entry = store.entry(KEY, ThemeMode.SYSTEM, { name -> ThemeMode.entries.find { it.name == name } }, { it.name })
    override val mode: StateFlow<ThemeMode> = entry.value
    override fun set(mode: ThemeMode) = entry.set(mode)
}

private const val KEY = "theme"
```

- **One per concern** (theme, language, consent, onboarding seen), never raw `Store` keys spread across features. A repository over it when a feature needs more than the value.
- **Interface + `Default` implementation**, so tests and previews pass a fake or a `MemoryStore()`.
- **Keys private**, the default in the `entry(...)` call. `entry(key, default)` exists for strings and booleans; `entry.reset()` removes the key.
- **`StateFlow`s**, so screens update live: a ViewModel's `load()` collects them like any repository flow.
- On web the value is `<app>.<key>` in localStorage, so `index.html` can read the theme before the first paint.
- Tests: `MemoryStore()`, or `SettingsStore(MapSettings())` from `multiplatform-settings-test`.

## What goes where

| Data | Store |
|---|---|
| theme, language, "seen" flags, consent choice | plain settings |
| tokens, session, anything secret | encrypted storage (see `feature-auth`) |
| domain data | the database |

## Theme mode

`SYSTEM / LIGHT / DARK` persisted; the root reads it and passes `dark` to the app theme.

## Language

See `feature-localization` for applying the chosen language.

## Logout

Per-user settings (consent, last-viewed, drafts) are cleared on logout; device settings (theme, language) stay.

## Settings screen

Each row is sentence case, says what it changes, and applies immediately (no save button). A destructive row (delete account, sign out) confirms in a dialog.
