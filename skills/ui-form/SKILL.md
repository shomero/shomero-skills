---
name: ui-form
description: Build a create or edit form — fields in the UiState outside any ScreenState, a derived canSubmit, a double-submit guard, blank optionals sent as null, a submit that catches the failure into a localized error, one screen for create and edit filled once from load(), design-system picker fields, and the saved row reaching the opener through its repository. Use when adding any create/edit screen, input validation, date or dropdown pickers, or returning to a previous screen after saving.
---

# Feature: create / edit form

## State

```kotlin
data class NoteFormUiState(
    val form: NoteForm? = null,                      // null until an edit has loaded its row
    val isSubmitting: Boolean = false,
    val error: ScreenState.Failure? = null,           // Offline or Error(reason), shown above the submit button
    val action: NoteFormUiAction? = null,
) { val canSubmit get() = form?.title?.isNotBlank() == true && !isSubmitting }

data class NoteForm(val title: String = "", val body: String = "", val dueDate: LocalDate? = null)
```

## Load, once

```kotlin
override fun load() = (editingId?.let { id -> repository.note(id).filterNotNull().map { it.toForm() } } ?: flowOf(NoteForm()))
    .onEach { loaded -> update { copy(form = form ?: loaded) } }   // never overwrite what the person typed
    .catch { e -> update { copy(error = e.toFailure()) } }
```

## Submit

```kotlin
private fun submit() {
    val s = state.value
    if (!s.canSubmit) return                                         // guards double taps too
    update { copy(isSubmitting = true, error = null) }
    viewModelScope.launch {
        try {
            val saved = repository.save(editingId, s.form!!.toRequest())
            update { copy(isSubmitting = false, action = NoteFormUiAction.Saved(saved.id)) }
        } catch (e: Exception) {
            if (e is CancellationException) throw e
            update { copy(isSubmitting = false, error = e.toFailure()) }
        }
    }
}
```

- Trim; blank optional → `null`, so the server applies its defaults.
- `toFailure()` gives `Offline` or `Error(reason)`; the screen turns it into a string resource, never the exception or response body. The typed fields stay.
- Field-level validation messages live in state as resources (`titleError: StringResource?`), computed in the ViewModel, each shown under its own field.

## Screen

The screen handles `Saved` by calling its `onSaved(id)` lambda, then `ClearAction` (see `compose-screen`). The app's lambda goes back. The opener's `load()` observes the same repository, so it shows the new row by itself: no result bus.

## Create and edit share one screen

- One route with `editingId: NoteId?`, passed as `@InjectedParam`. `load()` fills the form once (`form ?: loaded`), so a reload never wipes typing.
- Partial updates send only changed fields (`explicitNulls = false` on the body).
- A long form keeps a draft that survives process death (see `saved-state`).

## Pickers

- Shared `PickerField` (clickable surface: label, value or placeholder, trailing icon) and `DatePickerField` (wraps it, owns a local `showDialog`, shows the M3 `DatePickerDialog`) live in the shared components. A third copy of a picker means lift it (Rule of Three).
- **Date picker millis are UTC midnight**: `Instant.fromEpochMilliseconds(ms).toLocalDateTime(TimeZone.UTC).date`. Using the system zone shifts the day west of UTC.

## Wide windows

Show the form as a dialog-sized surface (capped, e.g. 640×760 dp) through a form-dialog scene, as a pushed screen on phones. Same route, metadata chosen by the app (see `ui-sheet-dialog`).

## Tests

`canSubmit` per field, double submit sends once, blank optional becomes null, failure keeps the fields and sets the reason, an edit's reload never overwrites typing.
