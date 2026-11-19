---
name: ui-react-web
description: Use React instead of Compose as the web UI while the Kotlin ViewModels, use cases and data layer stay shared — for when the site doesn't feel like a website (no text selection, find in page, real links, autofill, SEO, screen readers, a multi-megabyte first load). A Kotlin/JS bridge module exporting one non-generic controller per screen with TypeScript definitions, a useSyncExternalStore hook, ViewModel lifecycle tied to the component, and the module split it needs. Use when replacing Compose on web with React, or wiring React to shared Kotlin code.
---

# React on the web, Kotlin underneath

Compose on the web draws on a canvas, so browser behaviour is emulated. When users notice, swap the UI layer and keep everything below the ViewModel. It's React **or** Compose per web app: a Compose canvas inside React pages scrolls, selects and resizes wrong.

## Shape

```
features/notes/presentation   ViewModels, UiState, UiEvent: no Compose
features/notes/ui             Compose screens for the other platforms
apps/web-bridge               js library: one @JsExport controller per screen
web/                          Vite + React + TypeScript, imports the bridge as an ES module
```

- **Every module under the ViewModel needs a `js()` target**, and so does each library it uses (Ktor, Koin and androidx `lifecycle-viewmodel` have one; SQLDelight needs its web worker driver). Check before promising the swap.
- **Split `presentation` from `ui`**: if screens and ViewModels share a module, Compose lands in the JS bundle.

```kotlin
kotlin { js { browser(); binaries.library(); generateTypeScriptDefinitions(); useEsModules() } }
```

Kotlin/JS, not Kotlin/Wasm, for the bridge: Wasm's TypeScript interop stops at primitives and opaque handles. Build with `./gradlew :apps:web-bridge:jsBrowserProductionLibraryDistribution`; `web/package.json` depends on `"shared": "file:../apps/web-bridge/build/dist/js/productionLibrary"`.

## One controller per screen

`MVIViewModel<State, Event>` is generic and exposes a `StateFlow`; neither crosses `@JsExport` usefully. Export a thin, non-generic controller:

```kotlin
@JsExport
class NotesController {
    private val store = ViewModelStore()
    private val vm = ViewModelProvider.create(store, viewModelFactory { initializer { KoinPlatform.getKoin().get<NotesViewModel>() } })[NotesViewModel::class]
    private val scope = MainScope(); private val listeners = mutableSetOf<() -> Unit>()
    private var last = vm.uiState.value; private var snapshot = last.toView(); private var collecting: Job? = null
    fun getSnapshot(): NotesView = snapshot
    fun subscribe(listener: () -> Unit): () -> Unit {
        listeners += listener
        collecting = collecting ?: scope.launch(start = CoroutineStart.UNDISPATCHED) {
            vm.uiState.collect { s -> if (s != last) { last = s; snapshot = s.toView(); listeners.toList().forEach { it() } } }
        }
        return { listeners -= listener; if (listeners.isEmpty()) { collecting?.cancel(); collecting = null } }
    }
    fun search(query: String) = vm.handleEvent(NotesUiEvent.SearchChanged(query))
    fun clearAction() = vm.handleEvent(NotesUiEvent.ClearAction)
    fun dispose() { scope.cancel(); store.clear() }
}
```

- **The snapshot is an exported class built from the UiState**: ids as `String`, numbers as `Int`/`Double` (`Long` needs the BigInt flags), enums, lists as `KtList` read with `asJsReadonlyArrayView()`.
- **Same object until the state changes**: React compares snapshots with `Object.is`; a new object per call re-renders forever.
- **Collect `uiState` while React subscribes**, as `collectAsStateWithLifecycle()` does in Compose: that's what runs the ViewModel's `load()`, and it stops when the component unmounts. `UNDISPATCHED`, so an update made before the collector starts isn't missed.
- **One method per event**: sealed classes don't export cleanly. The action is a tag plus payload on the snapshot.
- **No `suspend` and no `Flow` in exported code.** Suspend export (`-Xenable-suspend-function-exporting`) is experimental; don't depend on it.
- The ViewModel's state doesn't survive a page reload: what must, goes in the URL.

## React side

```ts
import { useState, useEffect, useCallback, useSyncExternalStore, type DependencyList } from 'react'
export interface Controller<S> { getSnapshot(): S; subscribe(l: () => void): () => void; dispose(): void }
export function useController<C extends Controller<unknown>>(create: () => C, deps: DependencyList) {
  const [c, setC] = useState<C>()
  useEffect(() => { const next = create(); setC(next); return () => next.dispose() }, deps)
  const subscribe = useCallback((l: () => void) => c?.subscribe(l) ?? (() => {}), [c])
  return [useSyncExternalStore(subscribe, () => c?.getSnapshot() as ReturnType<C['getSnapshot']> | undefined), c] as const
}
```

- Created in the effect, so StrictMode's mount, unmount, mount never uses a disposed controller. Render a skeleton while it's `undefined`.
- Handle `snapshot.action` in a `useEffect`, then `c.clearAction()`, as `compose-screen` does in the screen.
- React Router owns URLs; nav actions call `navigate()`. The gate type (see `feature-auth`) applies unchanged.
- Strings: compose resources don't reach React. Generate the web i18n JSON from the same source in the build, every language.

## Rules

- Tests: Kotlin logic stays in `jvmTest`; the controller gets a `jsTest`; React components use Vitest + Testing Library.
- SPA by default. Server rendering only for SEO, and then the bridge must not touch `window` at import and needs a `nodejs()` target too.
- Not kotlin-wrappers (React written in Kotlin): the point is React developers and the React ecosystem.
