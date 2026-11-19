---
name: ui-list-detail
description: Build a searchable list that opens a detail screen, adaptive to window size — cache-first list ViewModel whose load() observes the cache and syncs once, search and filter that follow the state, a pure filtering use case, a detail ViewModel loaded by id, and on wide windows a resizable list-detail pane inside the tab instead of a pushed screen. Use when adding a list of entities, search/filter, a detail screen, or a two-pane layout.
---

# Feature: list → detail

## List ViewModel

- `load()` merges the repository's cache flow with one `sync*()`; the sync's failure goes through `failed(e)`, which keeps the cached list under a banner. Never `init {}`.
- `query` and `filter` are plain state fields; the list follows them with `state.map { it.query to it.filter }.distinctUntilChanged().flatMapLatest { … }` through a `FilterXUseCase`, so a newer query cancels the older one. Keep `totalCount` unfiltered for the "N of M" label.
- `Refresh` → `reload()`: `loadingAgain()` marks the content refreshing for pull-to-refresh; failure keeps the cached list and shows a retry, never empties the screen.
- Empty states are distinct: **no data yet**, **no match for the query**, **error with nothing cached**.
- `LazyColumn(items, key = { it.id })`: the id must never be defaulted (see `data-source`).

## Detail ViewModel

- Takes the id as `@InjectedParam` (`koinViewModel { parametersOf(id) }`); `load()` observes `repository.note(id)` and syncs it.
- **One ViewModel per id.** A side pane that switches rows keys it by id (`koinViewModel(key = id.value) { parametersOf(id) }`), or it shows the previous row.
- Missing row after sync → a "not found" state, not an endless spinner.

## Adaptive layout

- **Compact width**: tapping a row pushes the detail key (full window, own back button).
- **Expanded width**: list and detail share the tab entry as a resizable pane:
  - `selectedId` and `listFraction` are `rememberSaveable` UI state, not nav keys.
  - Tapping the selected row again closes the pane.
  - Keep `lastShownId` so the detail still renders while the pane animates closed.
  - Divider drag clamped (30–75%), drag sign flipped in RTL.

```kotlin
Box(Modifier.width(detailWidth).fillMaxHeight().clipToBounds()) {
    Box(Modifier.requiredWidth(targetDetailWidth).fillMaxHeight().align(Alignment.TopStart)) { detail() }
}
```

`requiredWidth`, not `width`: the shrinking parent would clamp `width` and reflow the content mid-animation.

## Long lists (pagination)

```kotlin
val shouldLoadMore by remember { derivedStateOf {
    val info = listState.layoutInfo
    val last = info.visibleItemsInfo.lastOrNull()?.index ?: 0
    info.totalItemsCount > 0 && last >= info.totalItemsCount - 3 } }
LaunchedEffect(shouldLoadMore) { if (shouldLoadMore) handleEvent(LoadMore) }
```

- The ViewModel keeps `nextPageToken`, `hasMore` and `isLoadingMore` in its state and ignores `LoadMore` when `!hasMore || isLoadingMore`.
- **Failures must reach the error state.** A repository `.catch { emit(emptyList()) }` shows "nothing here" when the network is down.
- A refresh is `reload()`: it cancels the running `load()` before starting again, so items never duplicate. Never launch a second collector by hand.
- Inserts (headers, footers, "all caught up") are built into the list in the ViewModel, never spliced in the screen.

## Navigation gotchas

- Ignore a push of the route already on top (`navigator.lastOrNull() != key`): a double tap creates a duplicate entry, which crashes Navigation 3 composition.
- A cached single-instance scene object means `NavDisplay` never sees the scene change: the detail renders nothing. Scenes are value objects with `equals`/`hashCode`.
- Expose the list as a public screen so the app's `App.kt` can host it under a tab.

## Tests

Filter use case (empty, no match, case, each filter), list VM (load, a newer query wins, sync failure keeps the cache), detail VM (not found, offline).
