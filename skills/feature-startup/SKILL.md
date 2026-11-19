---
name: feature-startup
description: App startup on com.shomero.kmp:startup — each piece of startup work an Initializer, listed in order in one @Single fun startup(): StartupCoordinator by @Named (Koin promises no order for getAll), started by initKoin on an app-scoped scope, Android's splash and the web page loader held on isLoading, timeouts so one SDK never freezes the splash, and heavy clients built off Main. Use when adding an SDK that needs initialising, a splash screen, or fixing slow cold starts or startup ANRs.
---

# Feature: startup

## An initializer

```kotlin
@Single @Named("session")
class SessionInitializer(private val session: SessionRepository) : Initializer {
    override val timeout = 5.seconds

    override suspend fun initialize(): Result<Unit> {
        session.restore()
        return Result.success(Unit)
    }
}
```

- `initialize()` returns a failure or throws; the coordinator logs either and moves on. It never waits longer than the initializer's `timeout` (or its own, 10 s), then cancels it. Worst case the app starts signed out or without an SDK.
- Override `isLoaded()` only when `initialize()` starts work that finishes later; the coordinator polls it.

## The order, in one place

```kotlin
// apps/<app>/composeApp: AppModule
@Single
fun startup(
    @Named("session") session: Initializer,
    @Named("analytics") analytics: Initializer,
): StartupCoordinator = StartupCoordinator(initializers = listOf(session, analytics))
```

- Take each initializer by name: never `getAll<Initializer>()` or an injected `List<Initializer>`, since Koin promises no order (analytics wants the user first).
- Monitoring isn't an initializer: it starts with Koin (see `feature-monitoring`).

## Start, and wait for it

```kotlin
fun initKoin(store: Store, target: ServerTarget, debug: Boolean) {
    val koin = startKoin<KoinApp> { modules(module { single { store }; single { target } }) }.koin
    CoroutineScope(SupervisorJob() + Dispatchers.Default).launch { koin.get<StartupCoordinator>().start() }
}
```

- Platform entry points only make what only they know (the `Store`, the server, `debug`), call `initKoin`, then show `App()`.
- **Android**: `installSplashScreen().setKeepOnScreenCondition { startup.isLoading.value }`, with `startup` from `by inject()`. A plain `StateFlow.value` read, cheap every frame; resolving the graph there runs constructors on Main.
- **Compose** (desktop, iOS, web): `val starting by koinInject<StartupCoordinator>().isLoading.collectAsStateWithLifecycle()`, and a loading screen while it's true.
- **Web**: keep `index.html`'s page loader up while it runs (see `web-page-loader`).
- Desktop: set the app name property before AWT starts; open at an expanded size.

## Heavy clients

- Build SDK clients `by lazy(SYNCHRONIZED)` and first touch them off Main; observe them with `flowOn(dispatcher)`. A backend client can cost 0.5–4 s to create.
- Anything that boots reflection (embedded servers in debug tools) starts on Default, not Main.
- Measure: monitor cold start per version (see `feature-monitoring`).

## Tests

The coordinator is tested in the library. Test each initializer: it reports a failure instead of hanging, and stops when cancelled.
