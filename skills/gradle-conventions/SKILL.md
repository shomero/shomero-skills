---
name: gradle-conventions
description: Gradle setup for a many-module KMP app — an included build-logic build of precompiled convention plugins, module build files that hold only plugin ids and dependencies, build files renamed after their module, typesafe project accessors, a version catalog, and gradle.properties tuned for KMP. Use when adding a module, a target, a plugin, a dependency, or when the build is slow or runs out of memory.
---

# Gradle conventions

## Layout

- `build-logic/` is an `includeBuild` holding precompiled script plugins (`src/main/kotlin/<app>.kmp.library.gradle.kts`). The file name is the plugin id.
- A module build file contains **only** plugin ids, dependencies, and what is truly module-specific.
- `settings.gradle.kts`: `enableFeaturePreview("TYPESAFE_PROJECT_ACCESSORS")` (`projects.services.notes`), and a recursive rename so each build file is `<module-name>.gradle.kts`.
- One `gradle/libs.versions.toml`; bundles for groups always used together (nav3, coil, ktor).

## Plugin ladder

| Plugin | Adds |
|---|---|
| `<app>.kmp.library` | KMP targets, Android library block, JVM target, `-Xexpect-actual-classes`, namespace derived from the project path |
| `<app>.kmp.compose` | Compose Multiplatform, compiler, hot reload, common Compose deps |
| `<app>.koin` | Koin compiler/KSP with the standard config |
| `<app>.compose.resources` | resources packaging, fixed `packageOfResClass` |
| `<app>.kmp.service` | library + koin (+ serialization) |
| `<app>.kmp.feature` | compose + resources + koin (+ serialization) |
| `<app>.android.app`, `.desktop.app`, `.kmp.app` | app shells, flavors, signing, versioning (see `build-flavors`) |

Kover and serialization can stay per-module aliases when membership is a per-module decision.

## gradle.properties

```properties
org.gradle.configuration-cache=true
org.gradle.caching=true
org.gradle.parallel=true
org.gradle.jvmargs=-Xmx6g
kotlin.daemon.jvmargs=-Xmx6g
kotlin.native.cacheKind=none        # native cache runs out of heap on many-module iOS links
android.nonTransitiveRClass=true
```

## Rules

- Never a `build` segment in a build-logic package: `.gitignore`'s `**/build/` hides the sources from `git add`.
- Fast inner loop: compile one module (`:features:notes:compileKotlinJvm`), not the app.
- Tests: `./gradlew jvmTest`.
- Versions: `versionName` from `version.properties`, `versionCode` from CI's build number (1 locally).
- Windows CI scripts are ASCII-only: PowerShell 5.1 reads BOM-less UTF-8 as cp1252, and an em dash becomes a quote.

Needs the convention plugins from the example project.
