---
name: kmp-module-layout
description: How a Kotlin Multiplatform app is split into apps/, features/, services/ and libraries/, the dependency rules between them, where routes, screens, repositories, shared UI and the theme live, and the checklist for adding a module. Use when creating a module, deciding where a class lives, or reviewing an import that crosses layers.
---

# KMP module layout

```
apps/        platform shells + composeApp (DI root, theme, Route, root NavDisplay)
features/    screens and their ViewModels, feature use cases. Depend on services + libraries
services/    data + business rules for one domain area. Depend on libraries only
libraries/   generic code: components (shared UI), design-system (the theme). Depend on other libraries only
```

## Rules

- **Features never depend on features.** A feature exposes screens and takes navigation as lambdas; the app's `App.kt` maps each `Route` to a screen and fills the lambdas in (see `navigation3`). UI two features need moves to `libraries/components`; colors, type and tokens live in `libraries/design-system`.
- **Apps depend on features, not services.** A service arrives transitively. The one exception: the app is the DI root for network infrastructure (the single API client), because it owns the build config.
- **All backend and database access lives in `services/`.** A feature or an app never calls an endpoint or a query itself.
- **Screens are the only type pinned to a feature.** ViewModels, use cases and repositories may live in `services/` or `libraries/` when that domain owns them.
- **Use the libraries before writing your own**: `com.shomero.kmp:mvi`, `com.shomero.cmp:nav3-*`, `com.shomero.kmp:settings`, `com.shomero.kmp:logger`, `com.shomero.kmp:startup`, `com.shomero.cmp:localization`. Wrap them in `libraries/` only to add what the app needs.
- **Package = path.** `com.<app>.<libraries|services|features>.<module-name>.*`, dash becomes dot. No package may contain a `build` segment: `.gitignore`'s `**/build/` then hides new sources from `git add`.
- **A project that already has another layout keeps it.** Follow what is there, and never move modules unless the task asks for it: a move touches every import and hides the real change.
- **Breaking a rule is said out loud**: name the rule and the reason in the PR. With the `com.shomero.layout` plugin, the build's warning names the rule.

## Inside a module

```
apps/<app>/composeApp/
  App.kt                   the root: theme, language, NavDisplay mapping every Route
  navigation/              Route.kt (one sealed interface), RouteUrls.kt (web addresses)
  di/                      initKoin, AppModule (the startup order, library classes)
features/<name>/
  presentation/<screen>/   <Name>Screen, <Name>UiState, <Name>UiEvent, <Name>ViewModel (+ <Name>UiAction), components/
  domain/                  use cases only this feature needs
  di/<Name>Module.kt
services/<area>/
  data/repository/         <Area>Repository + Default<Area>Repository
  data/sources/remote/     <Area>RemoteDataSource + Default…, dto/ with mappers
  data/sources/local/      <Area>LocalDataSource + Default…
  domain/                  use cases every consumer of the area needs
```

## Adding a module: checklist

1. Include it in `settings.gradle.kts` under its tree.
2. Name its build file `<module-name>.gradle.kts` (a settings script renames build files to the module name so the IDE tabs are readable).
3. Apply exactly one convention plugin (`<app>.kmp.feature`, `<app>.kmp.service`, `<app>.kmp.library`); no KMP target or Android block by hand. The Android namespace is derived from the project path.
4. Add a Koin module scanning **only its own package** (see `koin-annotations`).
5. If it gets a `README.md`, link it from the root README's module map in the same change.

Your CLAUDE.md names the app's package root and its convention plugin ids.
