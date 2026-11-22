---
name: design-system
description: Structure a Compose Multiplatform design system — one AppTheme with light/dark (and contrast) schemes, typography, shapes, and spacing and motion tokens as extensions on MaterialTheme, theme mode persisted as a setting, layouts that glide across width limits, components that are stateless and slot-based with previews, icons as lazy ImageVectors, the Rule of Three for what moves in, and how a screen is built from it: tokens and roles instead of raw values, the project's components first, hover and press tints clipped to the shape, and selection only on text people copy. Use when adding a component, a color or token, an icon, or theming, or when building or reviewing a screen's look.
---

# Design system

`libraries/design-system` owns the look: the theme and its tokens. `libraries/components` holds the shared components and strings built on it. Features never define colors, type or reusable components.

## Theme

```kotlin
@Composable
fun AppTheme(darkTheme: Boolean = isSystemInDarkTheme(), brand: Color? = null, content: @Composable () -> Unit) {
    MaterialTheme(colorScheme = if (darkTheme) darkScheme(brand) else lightScheme(brand),
                  typography = AppTypography, shapes = AppShapes, content = content)
}
val MaterialTheme.spacing: Spacing @Composable @ReadOnlyComposable get() = LocalSpacing.current
val StandardEasing = CubicBezierEasing(0.2f, 0f, 0f, 1f)
```

- Schemes from Material Theme Builder; ship medium and high contrast if you define them, or delete them.
- **Extra tokens are extensions on Material's theme** (`MaterialTheme.spacing.md`, `MaterialTheme.motion`), so they read beside its own; constants like easings are top-level `val`s in one file. Never an `object` holder.
- **Motion tokens** (emphasized, standard, enter, exit as springs), never ad-hoc `tween(300)`.
- Per-flavor brand color passed in at the root (staging looks different from prod).
- Theme **mode** (system / light / dark) is a setting: a persisted `StateFlow<ThemeMode>` (see `feature-settings`).
- Platform chrome (status bar icon color) via one `expect` effect.

## Components

- Stateless, `modifier: Modifier = Modifier` first optional param, **slots** (`content`, `leading`, `trailing`) over boolean flags.
- One component per file, previews in light and dark, two widths, and RTL for anything directional.
- Text from parameters; the design system holds no feature strings.
- Layouts that change shape at a width limit use `com.shomero.cmp:fluid` (`FluidSplit`, `FluidGrid`, `fluidSpec()`), never an `if (wide) Row else Column`: crossing the limit glides and children keep their state.
- Shared CompositionLocals (snackbar, top bar height, the loading indicator) are defined here and provided at the app root. The screen states' loader is `LocalLoadingIndicator` from `com.shomero.cmp:mvi-states`: provide the app's once.

## Building a screen from it

- **No raw values in a feature**: no `Color(0xFF…)`, `16.dp` padding or `fontSize = 14.sp`. Colors, type, corners and spacing come from the theme, so a theme change restyles the whole app.
- **Pick the role, not the look**: `primary` for the one main action, `surfaceContainer` for a raised area, `error` for errors, `onSurfaceVariant` for secondary text. A color chosen because it looks right in light mode breaks in dark.
- **Status colors are tokens too** (done, active, waiting, attention), so a state looks the same everywhere and keeps its meaning under any theme.
- **A missing token is a change to the design system**, not a value in a screen: add it there (in the token file, when the theme is generated from one) and say so in the PR.
- **The project's components before Material's, Material's before your own.** Read `libraries/components` before writing a button, field, card, pill or empty state, and pass the `id` a component takes (see `ui-automation`).
- **Never restyle a component in a feature** (a custom shape, a new color) to make one screen differ: change the component, or ask in the PR.
- **Forms come from the project's form components**, never a raw `Card` and `OutlinedTextField`: a tinted card takes the theme's seed colour, and a form drawn by hand drifts from every other one (see `ui-form`).
- **Check every state a person can reach**: default, pressed, disabled, focused, error, loading, empty, and long text in the longest language shipped; in light and dark, with text passing WCAG AA contrast in each; right to left (see `feature-localization`); and at every width (see `compose-insets-adaptive`).
- **No other app's brand** from a reference: no logos, colors, illustrations or copy (see `design-references`). No font the project can't ship: it needs an OFL licence, or one the customer owns.
- Which component to use when is the project's call: its CLAUDE.md wins over this skill.

## Hover and press

- **Clip the tint to the shape.** `.clickable` after `.background(shape)`, or in a `Card`'s or `Surface`'s modifier, draws its hover and press tint as a rectangle past the round corners. Use the component's own `onClick`, or the project's clipped modifier.
- **Keep the tint off the edges.** A clickable row sits a few dp in from the dividers and the controls beside it, with round corners; a menu item sits in from the menu's edges.
- **One target per row.** A row that holds its own buttons makes only its text part clickable, so hovering a button doesn't light the whole row too.

## Text people can copy

- **Selection is opt-in.** No `SelectionContainer` around the app's frame, the screens or a whole dialog: a drag over a clickable row, a card, a rail or a header must not highlight its words. Rows, cards, rails, headers, labels and controls are never selectable.
- **Copyable text is the exception, and wraps itself** in a `SelectionContainer` where it is drawn: the agent's or a chat's messages, both sides (in the design system's message and bubble, one container across a message's paragraphs); ids people quote, such as a ticket's number or a task's key on its own page; addresses, links, commands and logs; and error details someone may paste into a ticket (the shared error text).
- **Not inside what opens something.** A key in a row that opens the task stays plain; the same key over the task's own page is copyable. A component that is shown both ways takes a `copyable` flag.
- **A dialog keeps selection only around what it shows to copy**, such as a link or a code, never around its whole content.
- **Controls still wrap their label in `DisableSelection`** (button, chip, tab), so a label inside copyable text is pressed, never selected. Text in a menu stays unselectable.

## What moves in

The third copy of a component in features (Rule of Three): `git mv` the best one into `libraries/components`, delete the others, repoint callers in the same change.

## Icons

Lazily built `ImageVector`s as top-level `val`s in one `Icons.kt` (`val BackIcon by lazy { … }`), Material Symbols style. Directional ones are auto-mirrored.

## Images

Coil 3 with the Ktor network fetcher; `crossfade`, and `memoryCacheKey(url)` so the same image shared across sizes is decoded once. Pick the right size variant from the model (`avatar.getSize(AvatarSize.List)`).
