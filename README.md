# Shomero skills

Skills for Claude Code that teach how to build Kotlin Multiplatform + Compose Multiplatform apps the Shomero way: the architecture, the Kotlin rules, the UI patterns, the technical features every app has, and the workflow from branch to merged PR.

Each skill is a folder with one `SKILL.md`, short and precise: the shape, the rules with their reason, a checklist. [`catalog.json`](catalog.json) lists every skill with its category and the example-project pieces it leans on. The installer CLI reads the catalog.

## Rules for a skill

- One topic, under ~80 lines. What to do, and why in the same line.
- Generic: placeholders (`<app>`, `ApiClient`, `AppTheme`), a neutral example domain (`Note`). Project specifics belong in that project's `CLAUDE.md`.
- Vendor-specific knowledge goes in `skills/third-parties/<vendor>/`.
- `name` in the frontmatter equals the folder name, and is unique across the repo.
- A skill that relies on a base class or helper the reader may not have lists it in `requires` in the catalog.

## Skills

| Category | Skills |
|---|---|
| Architecture | `kmp-module-layout`, `mvi-viewmodel`, `screen-states`, `usecase`, `repository`, `data-source`, `sqldelight-table`, `koin-annotations`, `navigation3`, `saved-state` |
| Kotlin | `kotlin-conventions`, `code-comments` |
| UI | `compose-screen`, `design-system`, `compose-insets-adaptive`, `compose-keyboard-mouse`, `compose-performance`, `ui-list-detail`, `ui-form`, `ui-sheet-dialog`, `ui-react-web`, `ui-swiftui-ios` |
| Features | `feature-auth`, `feature-auth-hard-gate`, `feature-auth-soft-gate`, `feature-startup`, `feature-network`, `feature-logging`, `feature-analytics`, `feature-monitoring`, `feature-settings`, `feature-localization`, `feature-deep-links` |
| Build | `gradle-conventions`, `build-flavors`, `r8-release`, `kmp-web-target`, `web-page-loader` |
| Testing | `testing-kmp`, `coverage` |
| Workflow | `open-pr`, `review-pr`, `ui-automation`, `probe`, `e2e-cases`, `design-references`, `worktree-emulator` |
| Third parties | `codemagic`, `firebase`, `google-sign-in`, `kotzilla`, `mixpanel`, `revenuecat`, `supabase` |

## The example project

Some skills describe code the reader's project may not have yet: a base class, a navigator, a dev tool. Those skills point to an example project that ships each piece, listed under `examples` in the catalog:

| Piece | Used by |
|---|---|
| `mvi` | `mvi-viewmodel`, `screen-states`, `probe`, `compose-screen`, `saved-state`, `ui-list-detail`, `ui-form`, `ui-react-web`, `ui-swiftui-ios`, `feature-auth-soft-gate`, `testing-kmp` |
| `navigation` | `navigation3`, `ui-list-detail`, `ui-sheet-dialog`, `feature-auth*`, `feature-deep-links` |
| `startup` | `feature-startup`, `feature-auth`, `feature-auth-hard-gate` |
| `logger` | `feature-logging` |
| `database` | `repository`, `data-source`, `sqldelight-table`, `feature-auth` |
| `network` | `repository`, `data-source`, `feature-network`, `testing-kmp`, `supabase` |
| `analytics` | `feature-analytics`, `testing-kmp`, `mixpanel` |
| `settings` | `feature-settings`, `feature-auth-soft-gate` |
| `localization` | `feature-localization` |
| `design-system` | `design-system`, `ui-form` |
| `automation` | `ui-automation`, `probe`, `e2e-cases`, `compose-screen` |
| `deeplinks` | `feature-deep-links` |
| `build-logic` | `kmp-module-layout`, `gradle-conventions`, `build-flavors` |
| `scripts` | `coverage`, `worktree-emulator`, `codemagic` |
| `web-bridge` | `ui-react-web` |
| `swiftui-bridge` | `ui-swiftui-ios` |

Skills with no `requires` stand on their own.

### What exists, and what to build

The Shomero libraries already ship most pieces, and `shomero-kmp-starter` wires them together:

| Piece | Library |
|---|---|
| `mvi` | `com.shomero.kmp:mvi`, `com.shomero.cmp:mvi-states`, `com.shomero.kmp:connectivity` |
| `navigation` | `com.shomero.cmp:nav3-navigator`, `com.shomero.cmp:nav3-browser` |
| `startup`, `logger`, `settings` | `com.shomero.kmp:startup`, `com.shomero.kmp:logger`, `com.shomero.kmp:settings` |
| `localization` | `com.shomero.cmp:localization` |
| `analytics` | `com.shomero.kmp:analytics`, `analytics-mixpanel`, `analytics-testing` |
| `automation` | `com.shomero.cmp:automation-context`, `com.shomero.cmp:probe` and `probe-noop` |

Still to build:

- **`mvi`**: restoring state after process death (it keeps state in memory only), and for a SwiftUI UI, a base on KMP-ObservableViewModel's `ViewModel`.
- **`navigation`**: overlay scene strategies (dialog, form dialog, bottom sheet) and the resizable list-detail pane. `Navigator` doesn't drop a duplicate push.
- **`design-system`**: `PickerField`, `DatePickerField` and motion tokens, in a project's `libraries/`.
- **`automation`**: probe's Android and iOS drivers (web and desktop work).
- **New**: `database`, `network` (each project keeps its own API client, as the starter does in `services/api`), the analytics codegen plugin, `deeplinks`, `build-logic` (flavors, identity enum), `scripts`, `web-bridge`, `swiftui-bridge`.

### Not covered yet

Push notifications, runtime permissions, feature flags and forced updates: nothing tested exists to learn from yet. Each needs a skill once the example project has it.

## License and contributions

[Apache License 2.0](LICENSE). Use the skills in any project, private or commercial, change them, and share them; keep the license and the [NOTICE](NOTICE) when you pass them on.

Changes are welcome: open a pull request. What you contribute is under the same license, as the license itself says (section 5), so there is nothing to sign.
