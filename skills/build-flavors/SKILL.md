---
name: build-flavors
description: Staging and production environments in a KMP app — Android stg/prod flavors with stacked .stg/.debug suffixes so every variant installs side by side, BuildKonfig per flavor, -Penv for desktop/iOS, secrets only in local.properties or CI, and one launcher icon per variant. Use when adding an environment value, a secret, a new app shell, or touching build config.
---

# Build flavors

## Shape

- Android `productFlavors` on an `environment` dimension: `stg` (**the default**) and `prod`.
- `stg` adds `applicationIdSuffix = ".stg"`, the `debug` build type adds `".debug"`. They **stack**, so all four variants install together: `app.stg.debug`, `app.stg`, `app.debug`, `app`.
- BuildKonfig exposes `flavor` and per-env values. The flavor is resolved in order: `-Penv=stg|prod`, then a task name containing `Prod`, else `stg`.
- Desktop and iOS have no flavors: pass `-Penv=prod`.
- All of it lives in a convention plugin; app build files make one call with the app's identity (`appAndroid(Role.ADMIN)`).

## One identity enum for several apps

When a repo ships several apps (roles, white labels), an enum is the single source of truth for each one's id, app name, installer name, deep-link host, dev ports and Windows upgrade GUID. Its `init` asserts they are unique: a shared GUID makes one app's MSI overwrite the other's.

## Secrets

- Values come from gitignored `local.properties` (`api.stgUrl`, `api.stgKey`, `api.prodUrl`, …); CI injects them from secrets.
- **Strip whitespace at build time**: a trailing newline in a key makes an `Authorization` header throw.
- **Fail loudly** when a required value is blank, instead of falling back to localhost.
- Never a literal in `buildConfigField`. Prod values never in a tracked file.
- Debug conveniences (prefilled test login) are disabled when the task name contains `Release`.

## Launcher icons per variant

- Flavor source set → **background colour** (`src/stg/res` overrides `src/main/res`).
- Build type source set → **DEBUG badge** on the foreground (`src/debug/res`).
- Different files, so they merge independently: no `src/stgDebug` needed.
- Per-flavor launcher label in `src/stg/res/values/strings.xml`.

## Caveat

`BuildKonfig.flavor` is one value per Gradle invocation: build one environment at a time.

Needs the flavor convention plugin from the example project.
