---
name: worktree-emulator
description: Give each git worktree its own Android emulator (an AVD named after its ticket), always address devices by serial, and delete the AVD when the worktree is done. Use before any step that installs, launches, drives or records the Android app, and when finishing a worktree.
---

# worktree-emulator

Parallel sessions sharing one emulator reinstall and tap over each other, and a setting one changes (animation scale, network delay) leaks to the others. One worktree, one emulator, one ticket.

## Start

```bash
scripts/emulator start            # prints the serial, e.g. emulator-5558, on the last line
```

- AVD named after the ticket from the branch name, else the worktree folder (`--name KEY-123` to force).
- Cloned from a base AVD's hardware and image (`--base`, default from an env var); starts **wiped**, so install the build and sign in.
- **Idempotent**: call it at the top of any step that needs a device.
- `--animations off` for UI flows; `--animations on` for anything measuring animation cost. `--headless` for no window.
- `serial`, `list`, `stop` (keeps the AVD), `delete`, `prune` (worktrees that no longer exist).

## Use the serial everywhere

```bash
adb -s emulator-5558 …
ANDROID_SERIAL=emulator-5558 ./gradlew :apps:app:androidApp:installStgDebug   # else Gradle installs on every device
```

## Rules

- **Never drive a device another worktree owns**, and never a person's phone. Several devices in `adb devices` is normal; never a bare `adb`.
- `emulator-5554` is never handed out: it's the one a person starts by hand.
- **Delete when done** (PR merged or closed): an AVD costs several GB of disk, a running one 2–4 GB of RAM.
- A new worktree has no gitignored files (`local.properties`, SDK config JSON). Copy them from the main checkout before building.

Needs the `scripts/emulator` helper from the example project.
