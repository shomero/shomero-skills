---
name: kmp-web-target
description: Ship a Compose Multiplatform app to the browser with wasmJs — ComposeViewport, SQLDelight on a web worker, StorageSettings, no-op actuals for mobile-only SDKs, address-bar routing, and the rendering traps. Use when adding or fixing the web target, or when a change must keep the web build green.
---

# KMP web target (wasmJs)

If users feel the Compose web app isn't a real website, see `ui-react-web`: React for the UI, the same Kotlin underneath.

## Entry

```kotlin
fun main() { initKoin(); ComposeViewport(document.getElementById("app")!!) { App() } }
```

Compose draws in its own `<div id="app">`, never in `body`: `ComposeViewport` empties the element it's given, which would delete the page's loader. For the loader and a seamless start, see `web-page-loader`.

Build: `./gradlew :apps:app:webApp:wasmJsBrowserDistribution`. When the web build is what reviewers click through, a change that breaks it blocks review: build it whenever you touch UI.

## Platform pieces

- **Database**: SQLDelight web-worker driver + sql.js. Webpack needs `webpack.config.d` fallbacks `{ path: false, fs: false, crypto: false }`. Web queries are async: use the async generated API (`generateAsync = true`).
- **Settings**: `StorageSettings` (localStorage). Tokens there are readable by any script on the origin; keep them short-lived.
- **Mobile-only SDKs** (in-app purchases, native sign-in, native media) get no-op actuals that report `isAvailable = false`; the UI hides what isn't available.
- **SDKs without a wasm build**: call their REST API with Ktor instead (an auth provider's REST endpoints work everywhere).
- Split a library into separate modules when one target can't compile it, rather than `if (web)` in common code.

## Routing

Address after `#` follows the screen; the whole stack is stored on each history entry, so reload, shared links, Back and Forward work (see `navigation3`).

## Traps

- Unicode symbols (check marks, arrows) often render as empty boxes: use Material components (`Checkbox`, `RadioButton`) or icons.
- Fonts: bundle them as compose resources; system fonts differ per browser.
- No `Dispatchers.IO`, no blocking calls, no threads.
- Layout: decide with window size, not `isWeb` (see `compose-insets-adaptive`).
