---
name: navigation3
description: Navigation 3 in a KMP Compose app on com.shomero.cmp:nav3 — one @Serializable sealed interface Route : AppNavKey in the app module, requiresLogin and the Navigator's sign-in detour from rememberNavigator, every entry mapped in App.kt, features that take navigation as lambdas, arguments into ViewModels, and web addresses through routePath/routeStack and BrowserNavigation. Use when adding a destination, a route with arguments, a screen behind sign-in, or web addresses.
---

# Navigation 3

`com.shomero.cmp:nav3-navigator` (`AppNavKey`, `Navigator`, `rememberNavigator`) and `com.shomero.cmp:nav3-browser` (`BrowserNavigation`) on top of Navigation 3.

## One Route, in the app

```kotlin
// apps/<app>/composeApp/src/commonMain/kotlin/.../navigation/Route.kt
@Serializable
sealed interface Route : AppNavKey {
    @Serializable data object Notes : Route
    @Serializable data class Note(val id: NoteId) : Route { override val requiresLogin = true }
    @Serializable data object SignIn : Route
}
```

- **Every place is a case of this one interface.** Features declare no keys: they expose screens and take navigation as lambdas (`onOpenNote: (NoteId) -> Unit`), so no feature depends on another.
- `@Serializable`, with serializable fields only (ids, enums, strings).
- No sensitive value in a key: its name becomes the screen name in analytics and monitoring, and its fields the web address.

## App.kt maps each route

```kotlin
val backStack = rememberNavBackStack<Route>(Route.Notes)
val navigator = rememberNavigator(backStack, loginKey = Route.SignIn, isLoggedIn = { session != null })

NavDisplay(
    backStack = navigator.backStack(),
    onBack = navigator::back,
    entryDecorators = listOf(rememberSaveableStateHolderNavEntryDecorator(), rememberViewModelStoreNavEntryDecorator()),
    entryProvider = entryProvider {
        entry<Route.Notes> { NotesScreen(onOpenNote = { navigator.navigate(Route.Note(it)) }) }
        entry<Route.Note> { key -> NoteScreen(key.id, onBack = navigator::back) }
        entry<Route.SignIn> { SignInScreen(onSignedIn = { navigator.onLoginSuccess { target -> listOfNotNull(Route.Notes, target) } }) }
    },
)
```

`rememberViewModelStoreNavEntryDecorator()` gives each entry its own ViewModels, cleared when it leaves the stack. A screen's arguments reach its ViewModel as `@InjectedParam`, through `koinViewModel { parametersOf(id) }` in the screen's signature.

## Navigator

| Call | Does |
|---|---|
| `navigate(key)` | pushes `key`; signed out and `requiresLogin` → pushes sign-in and remembers `key` |
| `back()` | pops one, never the last |
| `reset(keys)` | replaces the stack: tabs, a flow that ends somewhere else, a hard gate |
| `popTo(key)` | drops what's above `key` |
| `onLoginSuccess { target -> stack }` | replaces sign-in with the stack for the remembered target |
| `onLogout()` | leaves only sign-in |

- **Always `navigate`**, never `backStack.add`: that skips the sign-in check.
- `navigate` doesn't drop a push of the key already on top, and a duplicate entry crashes Navigation 3: check `navigator.lastOrNull() != key` where a double tap is possible.
- Tabs: `reset(Route.Home, tab)`, one stack with Home at the root, so back from a tab lands on Home. Soft or hard gate: see `feature-auth`.

## Web: the address follows the screen

```kotlin
fun routePath(route: Route): String          // Note("42") → "/notes/42"
fun routeStack(path: String): List<Route>    // "/notes/42" → [Notes, Note("42")]: a typed address gets its parents

BrowserNavigation(
    history = history, stack = navigator.toList(), ready = !checkingSession,
    saveKey = ::routePath, restoreKey = { routeStack(it).last() },
    addressOf = { stack -> routePath(stack.last()) }, stackFor = ::routeStack,
    onRestore = { wanted -> if (session == null) navigator.navigate(wanted.last()) else navigator.reset(wanted) },
)
```

- Addresses go after `#`, and the whole stack is stored on each history entry, so reloads, links, Back and Forward work.
- `onRestore` goes through the navigator, so a shared link behind sign-in still passes through sign-in.
- `routePath` and `routeStack` are top-level functions in one file (`RouteUrls.kt`), tested so every route round-trips.

## Chrome and results

- A pushed screen takes the whole window; bar visibility comes from the topmost non-overlay key. Sheets and dialogs: `ui-sheet-dialog`. Predictive back: Nav 3's default `scaleOut(0.7f)` usually reads better as a fade.
- No result bus: a screen that changes data writes it through the repository, and the opener's `load()`, observing that repository, updates by itself.
