---
name: e2e-cases
description: Write a task's E2E cases as probe flows that people can read and that prove the change works — one case per file under probe/flows/<area>/, the task's key and sizes in the header, a seed step first when it needs an account, a name on every step, waits instead of sleeps, targets by automation-context ids, and the rule that a failing case is fixed in the app, not in the case. Use when a task lists E2E cases, when writing or fixing a probe flow, or when a flow fails after a change.
---

# e2e-cases

A case is one thing a person does and what they should see. It is written for a person and run by a machine, on every build from now on. The basics of ids and flows are in `ui-automation`; reading state and traces is in `probe`.

## One case, one file

```yaml
# probe/flows/notes/save-empty-title.yaml
# case: 3f1c… (Saving an empty title shows an error)
name: Saving an empty title shows an error
task: NOTE-14
sizes: [wide, phone]
steps:
  - name: Start on an account with notes
    seed: { fixture: account-with-notes }
  - name: Open the first note
    tap: { id: notes_screen_index_0_note_button }
  - name: Wait for the note
    waitForElement: { id: note_form_screen_title_text_field }
  - name: Empty the title
    setText: { into: { id: note_form_screen_title_text_field }, text: "" }
  - name: Tap Save
    tap: { id: note_form_screen_save_button }
    screenshot: true
  - name: See the error
    waitForElement: { id: note_form_screen_title_error_text }
  - name: The form stays open
    assertVisible: { id: note_form_screen_title_text_field }
```

- **Where:** `probe/flows/<area>/<slug>.yaml`, the area being the part of the app it is about, the slug its title in kebab case. One case per file, so a failure names one case.
- **Header:** `task:` with the task's key, `sizes:` with the case's sizes (`wide`, `phone`), and the case's id in a comment. The flow stays after the task is done and joins the project's suite.
- **Seed first.** A case that needs an account starts with `seed: { fixture: <name> }`, from the fixtures in `probe/env.yaml`. Each case gets its own fresh data, so cases never share data and can run in parallel. Never sign in through the form with a real account.
- **A name on every step,** for a person: `Tap Save`, `See the error`, beside the tool in the same step. The task's E2E tab shows these names next to a picture of each step; `screenshot: true` or `false` on a step overrides the default.
- **Wait, never sleep.** `waitForElement` for what appears, `waitForState` for a ViewModel that must settle, `assertVisible` and `assertNotVisible` only once the screen is there. A fixed `wait:` is a flaky case.
- **Target by id.** `--id` from `automation-context` first; a screen without ids gets them in the same change (`ui-automation`). Never coordinates, and text only for content the case itself typed.
- **Check what the person would see,** not internals: the text, the screen (`getCurrentRoute`), the state that drives it. One case checks one thing; more checks become more cases.
- **Edges are cases too:** nothing filled in, the server failing, no network, very long text, right to left, a phone width. Each one its own file, of the kind the task gave it.

## Run them

```bash
scripts/probe run-flow probe/flows/notes/*.yaml --url <served web build> --server <test server> --sizes wide,phone --retry 1 --report <dir>
```

- Against a debug build with probe in it, and the test server `probe/env.yaml` starts. Run the project's earlier flows too, not only the new ones.
- On a failure, read the report's failed step: its picture, the UI tree, the trace and the log say why before the code does.
- Done means every case passes at every size. A case that passes only on its retry is flaky: fix the race (usually a missing wait), don't leave it.

## Fix the app, not the case

- A failing case is a bug in the app until shown otherwise. Change the code until it passes.
- Change a flow only in how it reaches the screen: a wait, an id, a scroll. Never what it expects.
- When a case seems to expect the wrong thing, leave it and ask the person, saying why. The person decides what the product does.
- An earlier flow that fails because the task changes what it checks is updated, with the reason in the summary. Any other earlier flow that fails is a regression: fix the app.

Needs `automation-context` and `probe` (0.1.0 or later) from the example project; see `catalog.json`.
