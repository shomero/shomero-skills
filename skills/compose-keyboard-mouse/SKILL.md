---
name: compose-keyboard-mouse
description: Make a Compose Multiplatform screen work with a keyboard and a mouse as well as a finger — shortcuts for the frequent actions on the keys people know, Ctrl and Cmd both accepted, handled in onPreviewKeyEvent on KeyDown, text editing and the browser's own keys left alone, everything reachable by Tab with a focus ring, tooltips on icons with no words, and pointer icons that show what a thing does. Use when adding a shortcut, a tooltip, a pointer icon or focus handling, or when building a screen that ships to desktop or web.
---

# Keyboard and mouse

One Compose screen runs on a phone, a tablet, a desktop window and a browser tab. It must work with a keyboard and a mouse as well as with a finger. For using the width, see `compose-insets-adaptive`.

## Keyboard

- **Give the frequent actions a shortcut**, not every action, on the keys people know: Ctrl+S saves, Ctrl+F finds, Enter sends (Ctrl+Enter where Enter writes a new line), Escape closes a dialog, a menu or a search. Escape is not Back.
- **Accept Ctrl and Cmd**: `event.isCtrlPressed || event.isMetaPressed`, so one handler serves Windows, Linux and the Mac.
- **Handle a shortcut in `onPreviewKeyEvent`**, at the level that owns the action: the app's frame, the screen or the dialog. A focused text field takes its keys before `onKeyEvent` sees them.
- **Act on `KeyEventType.KeyDown` only, and return `true` only for a key you handled.** Otherwise the action runs twice, or typing breaks.
- **Leave text editing alone.** Never take over copy, paste, undo or select all in a field, and never delete an item on Backspace while someone types. Don't send on Enter while an input method is composing (Japanese, Chinese), or from a phone's on-screen keyboard.
- **A browser keeps some keys for itself**: new tab, new window, close tab and switch tab (Ctrl+T, Ctrl+N, Ctrl+W, Ctrl+Tab) never reach the page. Don't hang an action on those, or it is dead on web.
- **Everything works by keyboard alone**: Tab reaches each control in reading order, and the focused one shows it with the project's focus ring.

```kotlin
Modifier.onPreviewKeyEvent { event ->
    val save = event.type == KeyEventType.KeyDown && event.key == Key.S && (event.isCtrlPressed || event.isMetaPressed)
    if (save) onEvent(NoteFormUiEvent.Save)
    save                                                    // true only for the key handled
}
```

## Mouse

- **An icon with no words has a tooltip**: the project's component, else Material's `TooltipBox` with `PlainTooltip`. Never desktop's `TooltipArea`, which the other platforms don't have.
- **The tooltip's text is the icon's content description**, from the same string, plus the shortcut when there is one: "Save (Ctrl+S)". The icon keeps its content description.
- **A tooltip is never the only place** for a label, an instruction or an error. On a disabled control it says what is missing: "Select a row first".
- **The pointer shows what a thing does**: `Modifier.pointerHoverIcon(PointerIcon.Hand)` on links, buttons, tabs and navigation items while they are enabled, not on every clickable surface; `Crosshair` on a drawing surface; `Text` on text a custom view lets people select. Standard text fields and the design system's buttons already set theirs.
- **Grab and resize pointers are a different type on each platform**: add them to the design system with `expect` and `actual`, not in a feature (see `design-system`).
- Hover and press tints: see `design-system`.

## Checklist

- [ ] Checked at a phone's width and in a wide window
- [ ] Used once with only the keyboard: Tab order, focus ring, each shortcut, Escape
- [ ] Used once with only the mouse: tooltips on bare icons, pointer icons
- [ ] No shortcut on a key the browser keeps or a text field needs

Adapted for Compose Multiplatform from Google's `adaptive` skill (github.com/android/skills, Apache License 2.0).
