---
name: mvi-viewmodel
description: A screen's ViewModel on com.shomero.kmp:mvi — <Name>UiState and <Name>UiEvent, a @KoinViewModel extending MVIViewModel<State, Event>(), data from a cold load() flow written with update { }, ScreenState for each part that loads, reload() after a change, never init {} or LaunchedEffect, and one-time actions in the state. Use when adding or reviewing a ViewModel or its UiState/UiEvent files.
---

# MVI ViewModel

One state the screen draws, one door in: the screen sends `UiEvent`s to `handleEvent`, the ViewModel answers with a new `UiState`. The base is `MVIViewModel` from `com.shomero.kmp:mvi`.

## Files per screen

| File | Shape |
|---|---|
| `<Name>UiState.kt` | `data class`, every field defaulted. Each part that loads is a `ScreenState<T>`, starting at `Loading` |
| `<Name>UiEvent.kt` | `sealed interface` of what the person did |
| `<Name>ViewModel.kt` | below |
| `<Name>Screen.kt` | see `compose-screen` |

## Shape

```kotlin
data class NotesUiState(val notes: ScreenState<List<Note>> = ScreenState.Loading)

sealed interface NotesUiEvent {
    data object Retry : NotesUiEvent
    data class Delete(val id: NoteId) : NotesUiEvent
}

@KoinViewModel
class NotesViewModel(
    private val repository: NoteRepository,
    private val connectivity: Connectivity,
) : MVIViewModel<NotesUiState, NotesUiEvent>() {

    override fun initialState() = NotesUiState()

    override fun load() = repository.observeNotes()
        .onStart { update { copy(notes = notes.loadingAgain()) } }
        .onEach { list -> update { copy(notes = ScreenState.Content(list)) } }
        .retryWhenOnline(connectivity) { e -> update { copy(notes = notes.failed(e)) } }
        .catch { e -> update { copy(notes = notes.failed(e)) } }

    override fun handleEvent(event: NotesUiEvent) {
        when (event) {
            NotesUiEvent.Retry -> reload()
            is NotesUiEvent.Delete -> viewModelScope.launch {
                try { repository.delete(event.id); reload() }
                catch (e: Exception) { update { copy(notes = notes.failed(e)) } }   // failed() rethrows cancellation
            }
        }
    }
}
```

## Rules

- **Data comes from `load()` only.** It's a cold flow that runs while the screen collects `uiState`, stops five seconds after it leaves, and runs again when it comes back. Never fetch from `init {}` (it starts before anyone looks and can't run again) or from a `LaunchedEffect` (the UI then decides when data loads).
- **`reload()` after a change** the screen made, and for Retry. It restarts `load()` from the beginning.
- **Write only with `update { copy(...) }`**: it's atomic, so two events never lose each other's change.
- **Read `state.value` inside the ViewModel**, never `uiState.value`: `uiState` follows the state only while the screen collects.
- **Failures are exceptions.** `catch` on the flow, never `runCatching` (it catches the cancellation that stops `load()`, and shows an error each time the screen leaves). `failed(e)` gives `Offline` for `OfflineException`, `Error(reason)` for a `ReasonException`, `Error(Unknown)` otherwise, and keeps content already on screen under a banner. `retryWhenOnline(connectivity)` reloads once the network is back. Never put `e.message` in state. The rules for each state: see `screen-states`.
- **Several sources**: `merge(...)` them. Data that follows the state (a filter, the open item): `state.map { it.query }.distinctUntilChanged().flatMapLatest { ... }`, so a newer query cancels the older request.
- **Typed input stays outside `ScreenState`**, and a reload never overwrites it: `copy(form = form ?: loaded.toForm())`.
- **One-time effects** (navigate after a save, open a URL): an `action: <Name>UiAction?` field plus a `ClearAction` event; the screen handles it, then clears it (see `compose-screen`).
- **Dependencies**: repositories and use cases from the constructor, never the network client. Screen arguments arrive as `@InjectedParam` (see `navigation3`). Heavy derivation goes off Main with `flowOn(dispatcher)`, an injected `CoroutineDispatcher`.
- **Process death**: the library keeps state in memory and doesn't restore it yet. What must survive goes in the route, a local store, or saved UI state (see `saved-state`).
- **Analytics** for CTAs are mapped here from the event, ids only.
- **Shared state is not a shared ViewModel**: several screens that need one value observe the same repository flow.
- **A SwiftUI or React UI** drives the same ViewModel: see `ui-swiftui-ios` and `ui-react-web`.

## Checklist

- [ ] Data from `load()`: no `init {}`, no `LaunchedEffect` load, `reload()` after each change
- [ ] Each loading part is a `ScreenState`, failures through `failed(e)`
- [ ] No `HttpClient`, endpoint or database call: repositories and use cases only
- [ ] A test collects `uiState` and covers loading, content, offline, error and each event (see `testing-kmp`)

Needs `com.shomero.kmp:mvi` and `com.shomero.kmp:connectivity`; see `catalog.json`.
