---
name: compose-insets-adaptive
description: Edge-to-edge and adaptive layout in Compose Multiplatform — enableEdgeToEdge, who pads which inset, consumeWindowInsets to stop double padding, full-bleed media under the status bar, a navigation bar on phones vs a rail on wide windows with NavDisplay kept in movableContentOf, and a web layout. Use when a bar overlaps content, padding doubles, adding a full-screen media screen, or supporting tablets, desktop and web.
---

# Insets and adaptive layout

## Edge to edge

- `enableEdgeToEdge()` in the Activity; every screen handles its own insets.
- **The component that draws at the edge pads it**: the top bar adds `statusBarsPadding()`, the bottom toolbar `navigationBarsPadding()`, the text field area `imePadding()`.
- Nested scaffolds **double-pad** unless the outer one consumes what it applied:

```kotlin
Modifier.fillMaxSize().consumeWindowInsets(innerPadding).padding(innerPadding)
```

- Expose the top bar's real height (bar + status inset) as a `CompositionLocal` when content must start below it.
- Full-bleed media: content draws under the status bar; overlays get `statusBarsPadding()`; add a solid strip `Modifier.windowInsetsTopHeight(WindowInsets.statusBars)` where icons must stay legible.

For Android specifics, use the `edge-to-edge` Android skill if it is installed.

## Adaptive

| Width | Chrome | List → detail |
|---|---|---|
| Compact | bottom `NavigationBar` | push the detail |
| Medium / expanded | `NavigationRail` | two panes in the tab (see `ui-list-detail`) |

- Compute one `WindowSizeTier` at the root from `currentWindowAdaptiveInfo()`; features receive it.
- Inside a screen, a layout that changes shape at a width limit uses `com.shomero.cmp:fluid` (`FluidSplit`, `FluidGrid`, `fluidSpec()`), never an `if (wide) Row else Column`: the change glides and children keep their state.
- Put the `NavDisplay` in `movableContentOf`, so switching between bar and rail at a breakpoint keeps every screen's state.
- **Navigation is the project's shell**: screens never draw their own bar or rail.
- **A detail beside its list has no back arrow**, and one that fills the whole screen on a phone stops doing so on a wide window.
- **A list uses the room**: more columns on a wide window (`FluidGrid`, or `GridCells.Adaptive(minWidth)` in a lazy grid), not one column of stretched rows.
- **A title with actions is never one row with a weighted title**: on a phone the actions squeeze the title to nothing and their labels wrap. Below the width where both fit, the actions move to their own row under the title and wrap there. Check every screen with a header at 360 to 400 dp.
- Forms become dialog-sized on wide windows (see `ui-sheet-dialog`).
- Wide windows come with a keyboard and a mouse: see `compose-keyboard-mouse`.
- Desktop opens at an expanded size (e.g. 1280×800).

## Web

Top bar instead of a bottom bar; content centered at a readable max width (`widthIn(max = 840.dp)`) on a neutral background. Decide with the window size, not an `isWeb` flag, so a narrow browser gets the phone layout.
