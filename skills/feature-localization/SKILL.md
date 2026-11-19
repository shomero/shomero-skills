---
name: feature-localization
description: Localization in Compose Multiplatform on com.shomero.cmp:localization — AppLanguage.resolve(saved), ProvideLanguage(language) for the locale and right-to-left, rememberScriptFamily for non-Latin fonts on web, the navigator.languages hook in index.html, every string in composeResources in all ten languages (values-he identical to values-iw), a translation check before commit, logical start/end, mirrored icons and previews in both directions. Use when adding UI text, a screen, a language picker, or supporting a right-to-left language.
---

# Feature: localization

## The language, at the root

```kotlin
val saved by languages.saved.collectAsStateWithLifecycle()      // "" until the person picks one
val language = AppLanguage.resolve(saved.ifEmpty { null })      // the saved one, else the device's if the app speaks it, else English
val scriptFont = rememberScriptFamily(language)                 // web: the script's font; elsewhere null
ProvideLanguage(language) {
    AppTheme(fontFamily = scriptFont) { App() }
}
```

- `ProvideLanguage` sets the locale Compose resources read, the layout direction, and recomposes so every string is read again. Text and direction come from the same choice, so they never drift.
- Don't override the locale yourself (`AppCompatDelegate`, `AppleLanguages`, `Locale.setDefault`, `LocalLayoutDirection`): the library does it per platform.
- Save the choice as a setting (see `feature-settings`); the picker lists `AppLanguage.entries` with `nativeName` and `painterResource(language.flag)`.
- **Web**: add the library's `navigator.languages` script to `index.html`, before the app's script, or the switch does nothing there. A non-Latin script draws as boxes until its font has loaded, so keep the page's loader up until then.

## Strings

- **No UI text in Kotlin.** Every string lives in the module's `src/commonMain/composeResources/values/strings*.xml` and is read with `stringResource(Res.string.key)`.
- **Every new key ships in all ten languages** in the same change: English, Spanish, French, German, Portuguese, Chinese, Japanese, Hindi, Arabic and Hebrew. A missing locale falls back to English silently.
- **`values-he` and `values-iw` stay identical**: older Android reads Hebrew from `iw`.
- Strings several modules need live in `libraries/components`, not copied.
- Plurals: `<plurals>` + `pluralStringResource`. Arguments: `%1$s`, never string concatenation (word order differs per language).
- **Write `'` as a plain `'`.** Compose resources render `\'` literally.
- **Run the project's translation check script before committing**; it finds missing keys and `he`/`iw` drift.

## Layout direction

- **Logical edges only**: `padding(start =, end =)`, `Alignment.Start`, `Arrangement.Start`. Never `left`/`right`, never absolute offsets.
- **Mirror directional icons** (back, forward, chevrons, send): `Icons.AutoMirrored.*`, or `Modifier.graphicsLayer { scaleX = -1f }` under `LayoutDirection.Rtl`.
- **Don't mirror** clocks, media controls, charts' time axes, or brand logos.
- Numbers, phone numbers and emails inside RTL text: wrap in `⁦…⁩` (LTR isolate) when they render reversed.
- **Preview both directions**:

```kotlin
@Preview @Composable
private fun NoteRowRtlPreview() = ProvideLanguage(AppLanguage.entries.first { it.rtl }) {
    AppTheme { NoteRow(sampleNote) }
}
```

## Copy

- Sentence case. Say why a field is asked; say "you can change this later" when true.
- Automation ids are language-independent; UI tests assert on ids or routes, never on translated text.
