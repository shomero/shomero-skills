---
name: compose-performance
description: Keep a Compose Multiplatform screen smooth while it scrolls, animates and loads — scroll and animation values read late in lambda modifiers, derivedStateOf for fast values, remember with every key, lazy lists with key and contentType, layout and drawing without a late frame or per-frame allocation, Modifier.Node for custom modifiers, effects that don't restart, heavy work off the screen, and speed judged on a release build. Use when writing a list, an animation, a custom layout or modifier, or when a screen janks, recomposes too much or freezes.
---

# Compose performance

Follow these in the UI you write. Audit a screen that already works only when the task is about speed.

## Keep fast changes out of composition

- **Read scroll and animation values late.** Use the lambda forms, `Modifier.offset { }`, `graphicsLayer { }` and `drawBehind { }`, and pass a child `() -> Float` instead of the value: the frame is laid out or drawn again without recomposing.
- **Turn a fast value into a slow one** with `remember { derivedStateOf { listState.firstVisibleItemIndex > 0 } }`: the screen recomposes when the answer changes, not on every pixel. From Compose 1.13, `computedStateOf` does such simple checks for less; it doesn't exist before, so look at the project's version first.
- **Don't wrap what is already cheap.** `list.size`, or a boolean from plain values, is a plain `val` or a getter: no `derivedStateOf`, and no `remember { mutableStateOf(x) }` that the next line overwrites.
- **Never write a state you already read in the same composition.** That backwards write recomposes on every frame, and can loop.
- **Give `remember` every input it uses as a key**, or it keeps a stale value. Use it for costly objects and for identity, not for every value: each one is stored, and its keys compared, on every recomposition.
- **A composable that is cheap to run beats one that is skipped.** Don't chase recomposition counts.

## Stability

- **Don't add `@Stable`, `@Immutable` or `ImmutableList` by reflex.** Strong skipping already skips a composable that gets the same instance. The annotation makes Compose call `equals()` instead, which on a model with lists costs more than recomposing.
- **Add one only when all of this is true**: the class comes from a module the Compose compiler doesn't see, it is small and flat with no collections, and its source builds a new, equal instance on every emission.

## Lazy lists

- **Every item has a `key` and a `contentType`.** The key is a `String`, `Int` or `Long` from the data; the content type lets Compose reuse a row's composition while scrolling.
- **`itemsIndexed` when a row needs its index**, never `list.indexOf(item)`, which searches the whole list for every row.
- **A short, bounded list (about ten rows) is a `Column` with a loop.** A lazy list costs more than it saves there; keys and content types are for lazy lists only.
- **Collect a flow once, above the list**, never in each row.
- **No `LaunchedEffect` in a row to record that it was seen**: rows are composed before they scroll into view. Use `Modifier.onVisibilityChanged`.
- **No `BoxWithConstraints` or `SubcomposeLayout` inside a row**: they compose their content during layout, which is slow in a list that scrolls.

## Layout and drawing

- **Don't place one thing from another's `onGloballyPositioned` kept in state.** It lands a frame late and composes twice. Measure and place both in one `Layout`; for an element's own bounds use `onPlaced` or `onLayoutRectChanged`.
- **No wrapper `Box` or empty `Canvas` only to decorate**: put the modifier on the element, or draw with `drawBehind`.
- **Build a `Path`, a `Brush` or a shader in `drawWithCache`**, not in the draw block, where it is allocated again on every frame.
- **A custom modifier is a `Modifier.Node`** with its `ModifierNodeElement`, never `Modifier.composed { }`, which runs a composable for every element it is on.

## Effects

- **Nothing with a side effect runs in a composable's body.** `LaunchedEffect` is for suspending work, `SideEffect` for a plain call after composition.
- **A long-running effect reads a callback or a parameter through `rememberUpdatedState`**, or it keeps using the first one it got.
- **Don't change a `LaunchedEffect`'s key on every frame** of a scroll or an animation: each change cancels a coroutine and starts another.
- **Heavy work belongs below the screen** (see `usecase` and `repository`), off the main dispatcher. The web build has one thread, so a long loop freezes the page whatever dispatcher runs it: split it into steps with `yield()` between them.

## Judge speed on a release build

A debug build is much slower and proves nothing. Use Android's release build with R8 (see `r8-release`), and the web's `wasmJsBrowserDistribution`.

Adapted for Compose Multiplatform from Google's `jetpack-compose-performance` skill (github.com/android/skills, Apache License 2.0).
