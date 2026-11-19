---
name: koin-annotations
description: Dependency injection with Koin annotations in a KMP app — @Single(binds = ...)/@Factory/@KoinViewModel, Default<Interface> implementations, one @Module @ComponentScan @Configuration per module scanning only its own package, qualifiers, platform implementations annotated per source set instead of expect modules, the entry point's values as the only DSL, and tracing bindings by hand when compile safety is off. Use when adding a class to the graph, a module, or debugging NoDefinitionFoundException.
---

# Koin annotations

| Annotation | For |
|---|---|
| `@Single(binds = [X::class])` | repositories, data sources, clients: `DefaultX` bound to its interface `X` |
| `@Factory(binds = [X::class])` | anything built on a client that changes with sign-in (or it keeps the old session) |
| `@Factory` | use cases |
| `@KoinViewModel` | ViewModels |
| `@InjectedParam` | screen arguments, passed with `koinViewModel { parametersOf(id) }` |

## One module per Gradle module

```kotlin
@Module
@ComponentScan("com.<app>.services.notes")
@Configuration
class NotesModule
```

- **Scan only your own package.** A shared prefix (`com.<app>`) rediscovers every other module's classes: duplicate bindings and Kotlin/Native link-time signature clashes.
- `@Configuration` lets the app's `@KoinApplication` pick the module up without listing it.

## Rules

- **Never an `object` for behaviour.** Anything with state, I/O, a platform API, randomness or time is an interface with a `Default` class, injected through the constructor. An `object` only marks the `@KoinApplication`, anchors a class loader (`DesktopResources::class.java`), or is a `data object` case of a sealed type. A companion object holds its class's own constants and factory functions only.
- **Platform implementations**: no `expect object`, `expect fun` or `expect` module handing out a service. Each platform source set annotates its own implementation, and the module's `@ComponentScan` finds it in that platform's build. A platform that has none provides none: the caller takes the interface as nullable, and Koin passes null.
- **A library class that can't carry annotations** gets a `@Single fun` in the module that uses it (`fun startup(): StartupCoordinator`).
- **Values only the entry point knows** (the settings `Store`, the server, `debug`), and what the platform makes before Koin starts (an Android `Application`'s or an iOS app delegate's push handler), are the only definitions written in Koin's DSL, at `startKoin` inside `initKoin(...)`. What the system creates, like an Android activity, gets its dependencies with `by inject()`.
- **Never build a service with its constructor** in a ViewModel or a screen. When arguments are only known at run time, inject a factory interface and ask it.
- **Qualifiers** (`@Named` or a custom `@Qualifier`) disambiguate several bindings of one type, e.g. two `HttpClient`s. The name on the injection site must match the provider's.
- **Order needs a list you write**: never rely on `getAll<T>()` or an injected `List<T>` when order matters (startup); take each by `@Named` (see `feature-startup`).
- **App scope**: one `CoroutineScope(SupervisorJob() + Dispatchers.Default)` made in `initKoin`. Never `GlobalScope`.

## When compile safety is off

If `koinCompiler { compileSafety = false }`, a missing binding is a runtime `NoDefinitionFoundException`. For each new definition verify by hand:

1. Its package is inside its module's `@ComponentScan`.
2. Every constructor parameter has a binding reachable from the app.
3. The module is an `api`/`implementation` dependency of the app's `composeApp`.
4. Qualifiers match on both ends.

Start Koin with `initKoin()` before `App()` on every platform. If the project uses Kotzilla, `monitoring()` goes last in `startKoin`.
