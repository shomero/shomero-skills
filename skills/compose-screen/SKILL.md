---
name: compose-screen
description: How a Compose Multiplatform screen is built — <Name>Screen(viewModel = koinViewModel()) collects uiState with collectAsStateWithLifecycle() and draws a private <Name>Content(state, onEvent), navigation arrives as lambdas, ScreenState is drawn with ScreenStateContent, one-time actions are handled then cleared, and previews of Content cover light and dark, narrow and wide, and each screen state. Use when adding or reviewing a screen, a component, or a screen's previews.
---

# Compose screen

A screen draws a state and reports what the person did. It never fetches, and never decides when data loads.

When Compose doesn't feel native enough, the same ViewModels can drive React on web (`ui-react-web`) or SwiftUI on iOS (`ui-swiftui-ios`).

## Shape

```kotlin
@Composable
fun NotesScreen(
    onOpenNote: (NoteId) -> Unit,                       // navigation: the app fills it in
    viewModel: NotesViewModel = koinViewModel(),
) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    NotesContent(state, viewModel::handleEvent, onOpenNote)
}

@Composable
private fun NotesContent(state: NotesUiState, onEvent: (NotesUiEvent) -> Unit, onOpenNote: (NoteId) -> Unit) {
    Column(Modifier.fillMaxSize().automationContext("notes_screen")) {
        NotesHeader()
        ScreenStateContent(state.notes, onRetry = { onEvent(NotesUiEvent.Retry) }) { notes ->
            NoteList(notes, onOpen = onOpenNote)
        }
    }
}
```

A screen with arguments takes them first and passes them on: `viewModel: NoteViewModel = koinViewModel { parametersOf(id) }`.

## One-time actions

```kotlin
LaunchedEffect(state.action) {
    when (val action = state.action) {
        is NoteFormUiAction.Saved -> onSaved(action.id)
        null -> return@LaunchedEffect
    }
    viewModel.handleEvent(NoteFormUiEvent.ClearAction)   // always, or it fires again
}
```

## Rules

- **Collect with `collectAsStateWithLifecycle()`**, default minimum state. It is what starts the ViewModel's `load()` and stops it when the screen leaves or the app goes to the background.
- **`<Name>Content` is private** and takes the state, `onEvent` and the navigation lambdas: no ViewModel, no Koin, so previews and tests draw it with any state.
- **Navigation is lambdas** (`onOpenNote`, `onBack`), filled in by the app's `App.kt` (see `navigation3`). A tap that only navigates calls the lambda; navigation after an async result goes through `state.action`.
- **Draw loading parts as `ScreenState`**: a whole screen with `ScreenStateContent(state, onRetry, onLeave)`; a part of one with its own `when` and `LoadingState` / `FailureState`. `ScreenStateContent` fills the remaining height, so inside a scrolling column it disappears. The rules for each state: see `screen-states`.
- **No data from a `LaunchedEffect`.** Use it for one-time actions; its keys are what its body reads.
- **Render, never derive.** No `.filter {}`, sort, fallback or formatting in a composable; the ViewModel hands over render-ready values.
- **Read a neighbouring screen first.** The project's own patterns (state holders, navigation, DI) win over new ones, so every screen reads the same.
- **One screen per file**, named after it. Small private parts may stay beside it.
- **Server free text gets `maxLines` + `overflow`** inside a non-scrolling `Column` with a weighted sibling, or it pushes content off screen.
- **Params that skip.** Pass the same instance when nothing changed: strong skipping then skips the composable, with no `@Stable` or `ImmutableList` needed. A lambda-holding data class defeats skipping. More on smooth screens: see `compose-performance`.
- **A pushed screen takes the whole window**: no bottom bar, its own back button. Only tab destinations show the bar.
- **Automation ids** on the root (`automationContext("<name>_screen")`) and every element (see `ui-automation`), never with user data in an id.
- **Previews on `Content`**, in the same file: the app theme, light and dark, narrow and wide, one per screen state (loading, content, empty, error, offline), static sample state.

## Checklist

- [ ] `<Name>Screen(viewModel = koinViewModel())` collects with `collectAsStateWithLifecycle()` and draws a private `<Name>Content`
- [ ] Navigation as lambdas; actions handled and cleared
- [ ] Strings from resources, logical start/end (see `feature-localization`)
- [ ] Previews: light, dark, two widths, each screen state
