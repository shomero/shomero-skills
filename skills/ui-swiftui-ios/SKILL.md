---
name: ui-swiftui-ios
description: Use SwiftUI instead of Compose on iOS while Kotlin ViewModels, use cases and data stay shared — for when the iOS app doesn't feel like iOS. Start hybrid (a SwiftUI shell hosting Compose screens), then move screens to SwiftUI on shared ViewModels through KMP-ObservableViewModel and KMP-NativeCoroutines, map the MVI contract onto SwiftUI, and know why not SKIE or Swift export yet. Use when replacing Compose on iOS, adding a SwiftUI screen over shared Kotlin, or bridging StateFlow and suspend functions to Swift.
---

# SwiftUI on iOS, Kotlin underneath

## Step 1: a SwiftUI shell around Compose

Most "doesn't feel iOS" is the chrome: navigation bar, swipe back, tab bar, sheets, large titles. Put SwiftUI around Compose first:

```kotlin
fun NotesViewController(): UIViewController = ComposeUIViewController { AppTheme { NotesScreen(onOpenNote = {}) } }
```
```swift
struct NotesComposeView: UIViewControllerRepresentable {
    func makeUIViewController(context: Context) -> UIViewController { MainViewControllerKt.NotesViewController() }
    func updateUIViewController(_ vc: UIViewController, context: Context) {}
}
```

`NavigationStack` and `TabView` own navigation; each Compose screen is a destination. Stop here when it's enough. Move a screen to SwiftUI when its content still feels wrong: lists, text input, keyboard, scrolling.

## Step 2: SwiftUI screens on shared ViewModels

| Bridge | Status with Kotlin 2.4 | Use |
|---|---|---|
| KMP-ObservableViewModel 1.1 + KMP-NativeCoroutines 1.0.6 | stable, built on Kotlin 2.4.20, the one Kotlin's docs recommend | **default** |
| SKIE | lagged behind Kotlin 2.4.20 | only once a release supports your Kotlin version |
| Swift export | Alpha, direct integration only, breaking changes expected | when it reaches Beta |

```kotlin
plugins { id("com.rickclephas.kmp.nativecoroutines") version "1.0.6" }
kotlin {
    sourceSets.all { languageSettings { optIn("kotlinx.cinterop.ExperimentalForeignApi"); optIn("kotlin.experimental.ExperimentalObjCName") } }
    sourceSets.commonMain.dependencies { api("com.rickclephas.kmp:kmp-observableviewmodel-core:1.1.0") }
}
```

The iOS framework `export(...)`s the presentation modules. In Xcode, add the Swift packages `KMP-ObservableViewModel` (`KMPObservableViewModelCore`, `KMPObservableViewModelSwiftUI`) and `KMP-NativeCoroutines` (`KMPNativeCoroutinesAsync`), plus one file:

```swift
import KMPObservableViewModelCore
import shared
extension Kmp_observableviewmodel_coreViewModel: @retroactive ViewModel { }
```

## The base class must be the library's

SwiftUI re-renders only when the ViewModel extends `com.rickclephas.kmp.observableviewmodel.ViewModel` and its state is that library's `MutableStateFlow(viewModelScope, initial)`. So **`MVIViewModel` itself must extend it**; on Android it is still an androidx `ViewModel`. Otherwise SwiftUI silently never updates. `com.shomero.kmp:mvi` extends androidx's `ViewModel` today, so SwiftUI screens need the example project's `swiftui-bridge` variant of it.

Its `viewModelScope` is the library's `ViewModelScope`, not a `CoroutineScope`, and it hides androidx's. Import `launch`, `stateIn` and `MutableStateFlow` from `com.rickclephas.kmp.observableviewmodel`; anything that needs a real scope (`launchIn`, kotlinx `stateIn`) takes `viewModelScope.coroutineScope`.

The generic `uiState` reaches Swift untyped (Objective-C export drops interface generics), so each concrete ViewModel adds one typed property. Not `state`: that's the base class's own. Observing it collects `uiState`, which is what runs `load()`:

```kotlin
@NativeCoroutinesState val viewState: StateFlow<NotesUiState> get() = uiState
```

## MVI in SwiftUI

```swift
import KMPObservableViewModelSwiftUI

struct NotesScreen: View {
    @StateViewModel var viewModel = ViewModels().notes()          // Kotlin KoinComponent helper
    @Binding var path: [Route]
    var body: some View {
        List(viewModel.viewState.notes, id: \.id) { note in
            Button(note.title) { viewModel.handleEvent(event: NotesUiEventOpen(id: note.id)) }
        }
        .onChange(of: viewModel.viewState.action) { _, action in
            if let open = action as? NotesUiActionOpenDetail { path.append(.detail(open.id)) }
            if action != nil { viewModel.handleEvent(event: NotesUiEventClearAction.shared) }
        }
    }
}
```

- Sealed events and actions are cast with `as?`, without exhaustiveness checks (SKIE or Swift export add them later). Objects are `.shared`.
- **Navigation is SwiftUI's**: routes become a Swift `enum Route: Hashable`. `navigation3` doesn't apply to these screens.
- Keep the Kotlin API Swift-friendly: no generics in public signatures, no default arguments Swift must rely on (it sees every parameter), names that read well in Swift.
- Strings: SwiftUI doesn't read compose resources. Generate `Localizable.xcstrings` from the same source in the build, every language.
- Previews and tests: build the ViewModel from fakes through a Kotlin factory; Kotlin logic stays tested in `commonTest`.
