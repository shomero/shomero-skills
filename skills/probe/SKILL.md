---
name: probe
description: Go past taps with com.shomero.cmp:probe — run it pinned, in the background or headless, read ViewModel state and the trace when a screen looks wrong (dumpViewModels, getState, waitForState, dumpTrace), wait on a state in a YAML flow, the other gestures, how screens and ViewModels reach probe, and the system tools for what lies outside the app. Use when a screen shows a spinner that never stops or doesn't update, when a flow races a load, or when setting probe up in a project.
---

# probe

The basics (ids, `dumpUiTree`, `tap`, `setText`, flows, safety) are in `ui-automation`. This is the rest: `probe help` lists every tool.

## Run it

- **Use `scripts/probe` when the project has it**: it pins the version, so the CLI and the library in the app match.
- **A debug build with probe in it**, behind the project's own build flag (your CLAUDE.md names it). Web: build and serve it, then `probe web --url http://localhost:8080 &` (port 4250). Desktop: the app listens on 4251, so add `--port 4251`.
- **Run it in the background, or headless on Linux** (a virtual display), so its window never takes the person's focus.

## More gestures

`longPress`, `doubleTap`, `scroll --dy 400`, `swipe --from <id or x,y> --to …`. Target by `--id` first, since ids survive copy changes and every language; `--role button --name "Save"` or `--text …` when there is none, and then name the screen (see `ui-automation`).

## Read the state when the screen looks wrong

Before reading code:

- `probe dumpViewModels` lists the ViewModels alive now, whether each one's `load()` runs, and its state.
- `probe getState NotesViewModel` gives one state field by field (no name: the newest ViewModel, usually the screen on top).
- `probe waitForState NotesViewModel --field notes.refreshing --is false` waits until it reads that, 5 s at most.
- `probe dumpTrace` gives what happened, in order: screens shown, ViewModels made and cleared, each load's start and stop (`completed`, `cancelled`, or `failed: …`), and each state change with only the fields that changed. `--since <seq>` gives what came after.
- `probe watch` prints screen views and analytics events live, with each ViewModel change. A new screen must log its screen view.

| Symptom | In the trace |
|---|---|
| A spinner that never stops | a `loadStarted` with no state change after it, or a `loadFailed` |
| A screen that doesn't update | a state change that never came, or one in another ViewModel |

## In a flow

```yaml
  - tap: { id: notes_screen_refresh_button }
  - waitForState: { viewModel: NotesViewModel, fields: { notes.refreshing: false }, timeoutMs: 5000 }
  - assertNotVisible: { id: notes_screen_refresh_failed_text }
```

- **After an action that loads, check the state with `waitForState`, not `getState`**: `getState` reads once and races the load.
- A field is its dotted path as `getState` prints it, and values compare as text (`false`, `null`).
- Every step is a tool; besides those in `ui-automation`: `assertNotVisible`, `pressKey`, `waitForState`.

## How the app reaches probe

- **ViewModels** reach it through the MVI library's observer (`MviDebug.observer`, set once at startup): no ViewModel or screen changes for it.
- **States are their `toString()`.** Keep secrets out of state classes, or override `toString` to hide them.
- **A new screen goes in the app's list of screens for probe**, so the navigation map knows it.
- Nothing in a release listens or reports: it takes `probe-noop` (see `ui-automation`).

## Edges

- **`pressBack` refuses to leave the app.** If there's nothing to go back to, the screen added no history entry: often a navigation bug.
- **Outside the app** (another page, a popup, a browser dialog, a permission prompt) use the `system*` tools, after `armSystemAccess` for dialogs.

Needs `automation-context`, `probe` and `com.shomero.kmp:mvi` from the example project; see `catalog.json`.
