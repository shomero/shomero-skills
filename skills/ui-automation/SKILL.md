---
name: ui-automation
description: Name every element with com.shomero.cmp:automation-context ids, then see, drive and check the running app with com.shomero.cmp:probe (dumpUiTree, tap --id, setText, getCurrentRoute, dumpScreenViews) instead of screenshots and guessed coordinates, and leave a YAML flow behind with every screen change. Release builds take probe-noop. Use whenever you need to see, tap, type into, assert on or test live app UI, or add ids to a screen.
---

# ui-automation

Building only proves it compiles. Open the change and use it, the way a reviewer will.

## Automation ids

```kotlin
Column(Modifier.automationContext("home_screen")) {
    Text(title, Modifier.automationText("title"))                          // home_screen_title_text
    Button(onClick = onAdd, modifier = Modifier.automationButton("add")) { Text(add) }   // home_screen_add_button
    notes.forEachIndexed { i, note ->
        Row(Modifier.automationIndex(i)) { Text(note.title, Modifier.automationText("title")) }   // …_index_2_title_text
    }
}
```

- **The screen's root** gets `automationContext("<name>_screen")`, the screen's file name without `Screen`, in snake_case. A dialog or sheet gets `<name>_dialog`; a section that groups controls gets its own context, so two Save buttons differ.
- **Everything gets an id**: every element a person acts on (`automationButton`, `automationTextField`, chips, switches, tabs, rows that open something) and every text a check reads (`automationText`: titles, values, statuses, errors, empty states).
- **The name says what it's for**, in English snake_case: `save`, `email`, `sign_in_tab`. Never translated copy, a position, or user data.
- A design-system component that takes an `id` names itself: pass it, don't wrap it.
- **Name the whole screen**, not only the part your change is about, so nobody has to add ids later. A screen you change that has no ids gets them all in the same change; say so in the PR.
- A project with another id convention keeps it.
- **Check them** with `probe dumpUiTree`: every control shows its id in brackets.

## Drive it with probe

1. Run a debug build with `com.shomero.cmp:probe` and `ProbeRoot { … }` around the app. Web: `probe web --url http://localhost:8080 &`. Desktop: the app listens on 4251, add `--port 4251`.
2. **Read the screen** with `probe dumpUiTree`: each element's id, role, name, text, bounds and states. `probe getScreenshot` only to check how it looks.
3. **Act**: `probe tap --id home_screen_add_button`, `probe setText "Buy milk" --id note_form_screen_title_text_field`, `probe pressKey Enter`, `scroll`, `pressBack`. Read the tree again after each step.
4. **Check what the app reports**: `probe getCurrentRoute`, `getBackStack`, `dumpScreenViews`, `dumpAnalyticsEvents`, or `probe watch` live.
5. Check the error, offline and empty states, not only the happy path.

Reading ViewModel state and the trace, waiting on a state in a flow, more gestures and what lies outside the app: see `probe`.

## Flows (the deliverable)

```yaml
# probe/flows/add-note.yaml
name: Add a note
open: /
steps:
  - getCurrentRoute: { is: Notes }
  - tap: { id: notes_screen_add_button }
  - setText: { into: { id: note_form_screen_title_text_field }, text: Buy milk }
  - tap: { id: note_form_screen_save_button }
  - waitForElement: { id: notes_screen_index_0_title_text }
  - assertVisible: { text: Buy milk }
  - dumpScreenViews: { last: Notes }
  - getScreenshot: note-added
```

- Every change to a screen adds or updates a flow for its happy path plus back or close. Run it with `probe run-flow probe/flows/<file>.yaml --url <address>` (or `--port 4251`); it must pass before you finish.
- Wait with `waitForElement` (it polls up to 10 s), not `wait: <ms>`; `assertVisible` checks now. Credentials from the environment, never committed.

## Safety

- Release builds take `com.shomero.cmp:probe-noop`, the same API doing nothing. Never ship a build with probe in it.
- Probe binds `127.0.0.1` only. Never point it at production with real accounts.
- Store captures only in the scratchpad or a private page; never commit them.
- A tool probe lacks: add it to probe, don't work around it with screenshots or coordinates.

Needs `automation-context` and `probe` from the example project; see `catalog.json`.
