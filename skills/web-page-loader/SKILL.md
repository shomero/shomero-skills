---
name: web-page-loader
description: A seamless start for a Compose web app — index.html draws the app's loader from the first paint, Compose draws under it in its own element, and the loader fades once, onto a finished first screen. Use when writing a wasmJs app's index.html or main(), or when its start flashes a blank frame, a second loader, or a screen that fills in late.
---

# Web page loader

The wasm takes seconds to download. Visitors should see one loader from the first paint, then the finished first screen: no white page, no blank frame, no second loader, nothing popping in after.

## index.html

- An inline script in `<head>` picks the theme before the first paint (the saved choice, else `prefers-color-scheme`), sets `data-theme` on `<html>`, `color-scheme` and `theme-color`: a dark visitor never sees a white page.
- Draw the app's own loader in HTML and CSS, the same look and colors as the Compose one. A JVM test that reads `index.html` and compares its colors with `AppTheme` fails when they drift.
- Compose gets its own element; the loader sits above it and fades once:

```html
<body>
<div id="app"></div>
<div id="page-loader"><!-- the loader --></div>
<script>
  window.appReady = function () {
    var loader = document.getElementById("page-loader");
    if (!loader) return;
    loader.classList.add("done"); // opacity 0 over 200 ms
    setTimeout(function () { loader.remove(); }, 250);
  };
</script>
```

`#app` fills the page; `#page-loader` is `position: fixed; inset: 0` with the highest `z-index`.

## main()

```kotlin
fun main() {
    val root = document.getElementById("app") ?: return
    initKoin()
    ComposeViewport(root) { App(onReady = ::hidePageLoader) }
}

// Three frames on, so the screen is painted under the loader before it fades.
private fun hidePageLoader(): Unit = js("requestAnimationFrame(() => requestAnimationFrame(() => requestAnimationFrame(() => window.appReady && window.appReady())))")
```

Never `ComposeViewport(document.body)`: it empties the element it's given, which deletes the loader the moment the wasm starts.

## Ready means the first screen is complete

Not when Compose starts, and not when the session is known: when the first screen has its content.

1. **Before the first screen**, under the page loader: the session check, every string, and any script font. On web `stringResource` draws blank until its file has downloaded, so read every module's strings with `getString(environment, it)` in the same resource environment the screens use.
2. **Count loaders.** The loader every screen state draws (`LocalLoadingIndicator` with `com.shomero.cmp:mvi-states`) counts itself while it shows: a `LoadingStates` holder in a composition local, `DisposableEffect { shown++; onDispose { shown-- } }`.
3. **Data with no loader counts too.** A screen that waits without showing a loader (sign-in waiting on the server's sign-in options, say) calls `ReportLoading(waiting)`, so the loader doesn't fade onto half a screen.
4. **Wait for quiet.** `onReady` runs once nothing has been loading for 200 ms, so a loader that starts a few frames late still counts. Cap the wait at 5 s, so a part of the screen that keeps loading never holds the page back:

```kotlin
withTimeoutOrNull(5.seconds) {
    snapshotFlow { states.shown == 0 }
        .transformLatest { idle -> if (idle) { delay(200); emit(Unit) } }
        .first()
}
onReady()
```

## Images on the first screen

On web a resource image (`painterResource`) loads after its screen has drawn. An image on the first screen, such as a sign-in button's logo, is an `ImageVector` built in code: it's there on the first frame.

## Check it frame by frame

Screenshots taken every 100 ms or so miss a 100 ms flash. Record the load in headless Chromium with the `disabled-by-default-devtools.screenshot` trace category, which keeps every frame, and look at the frames around `appReady`: the loader, then the finished screen. To get real data into a local build, serve it with a small proxy that passes API calls on to a real server. Check signed out and signed in, and both a first and a cached load.

## Checklist

- [ ] Theme picked in `<head>` before the first paint; the HTML loader matches the Compose one.
- [ ] `ComposeViewport` gets `#app`, never `body`.
- [ ] Strings and fonts are read before the first screen.
- [ ] Every loader counts itself; a screen waiting without one calls `ReportLoading`.
- [ ] `onReady` waits for 200 ms with nothing loading, capped at 5 s.
- [ ] No resource image on the first screen.
- [ ] Traced frame by frame: one loader, then the finished screen.
