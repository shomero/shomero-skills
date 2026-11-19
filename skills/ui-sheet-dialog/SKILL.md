---
name: ui-sheet-dialog
description: Make bottom sheets and dialogs real Navigation 3 destinations — overlay scene strategies, a bottom sheet scene that animates out on back, a form-dialog scene for wide windows, metadata supplied by the app, and keeping the bottom bar steady under an overlay. Use when adding a bottom sheet, dialog, confirmation, or a detail shown as a sheet.
---

# Feature: sheet or dialog as a destination

## Strategies, in this order

```kotlin
NavDisplay(
    sceneStrategy = DialogSceneStrategy<Route>() then FormDialogSceneStrategy() then ModalBottomSheetSceneStrategy(),
    …
)
```

All three produce `OverlayScene`s: the screen below stays composed and visible.

## The bottom sheet scene

```kotlin
override val content: @Composable () -> Unit = {
    val state = rememberModalBottomSheetState(skipPartiallyExpanded = properties.skipPartiallyExpanded)
    sheetState = state
    ModalBottomSheet(onDismissRequest = onBack, sheetState = state) { entry.Content() }
}
override suspend fun onRemove() { sheetState?.hide() }   // back still plays the hide animation
```

- Give the scene **value-based `equals`/`hashCode`**, so an unchanged sheet doesn't re-animate on recomposition.
- Nav 3's `DialogScene` composes the entry bare; a full-size form would cover the window. A form-dialog scene wraps the entry in a capped `Surface`.

## The app chooses, the feature doesn't know

```kotlin
// App.kt
entry<Route.NoteForm>(metadata = if (isExpanded) FormDialogSceneStrategy.formDialog() else emptyMap()) { key ->
    NoteFormScreen(key.editingId, onSaved = { navigator.back() })
}
entry<Route.ConfirmDelete>(metadata = DialogSceneStrategy.dialog()) { key -> ConfirmDeleteDialog(key.id, onDone = navigator::back) }
```

The feature exposes a screen with lambdas; the app's `App.kt` maps the route and decides sheet, dialog or screen. Features stay unaware of window size.

## Rules

- **Sheet → next screen**: dismiss the sheet first, then push, so back lands on the screen, not the sheet.
- **Bar visibility from the topmost non-overlay key.** Keep the overlay set in one place (a marker interface beats a hand-listed set: a forgotten key hides the bar).
- An opaque per-entry background decorator stops screens bleeding through during transitions; exempt overlay keys from it.
- Confirmations that do something irreversible (delete, cancel) are dialog routes with their own ViewModel, which writes through the repository the opener observes, not a `remember { mutableStateOf(true) }` in the screen.
- Predictive back: replace Nav 3's default `scaleOut(0.7f)` with a fade if it looks wrong for overlays.

Needs the scene strategies from the example project's `navigation` piece; `com.shomero.cmp:nav3` doesn't ship them yet.
