---
name: testing-kmp
description: Unit-test a KMP app — ViewModel tests that collect uiState in backgroundScope (or load() never runs), fakes written as classes, FakeConnectivity for offline, the real repository → data source → client chain over Ktor's MockEngine where the wire matters, failures as OfflineException and ReasonException, and use-case boundary tests, run with jvmTest. Use when writing or reviewing any test, or adding a ViewModel, repository or use case.
---

# Testing KMP

`./gradlew jvmTest`, not `test`: `test` matches no KMP task and runs nothing.

## ViewModel

```kotlin
@BeforeTest fun setUp() = Dispatchers.setMain(UnconfinedTestDispatcher())
@AfterTest fun tearDown() = Dispatchers.resetMain()

@Test fun retryLoadsAgainAfterAnError() = runTest {
    val notes = FakeNoteRepository { throw ReasonException(ErrorReason.Server) }
    val viewModel = NotesViewModel(notes, FakeConnectivity())
    backgroundScope.launch(UnconfinedTestDispatcher(testScheduler)) { viewModel.uiState.collect {} }
    assertEquals(ScreenState.Error(ErrorReason.Server), viewModel.uiState.value.notes)

    notes.answer = { listOf(note("1")) }
    viewModel.handleEvent(NotesUiEvent.Retry)
    assertEquals(ScreenState.Content(listOf(note("1"))), viewModel.uiState.value.notes)
}
```

- **Collect `uiState` in `backgroundScope`**, or `load()` never runs. The collector stops with the test.
- Cover: `Loading` first, `Content` after a load, `Offline` on an `OfflineException`, `Error` on anything else, content kept on a failed reload, each event, `reload()` after a change.
- With `FakeConnectivity(online = false)`, check the screen reloads by itself when `online` turns true, and that Retry reloads even while it's false.
- Pass the test dispatcher wherever the code takes a `CoroutineDispatcher`; a hard-coded one escapes the scheduler.
- Analytics: assert the events each ViewModel sends with `RecordingAnalytics` from `com.shomero.kmp:analytics-testing`.

## Fakes, not mocks

```kotlin
class FakeNoteRepository(var answer: () -> List<Note> = { emptyList() }) : NoteRepository {
    val created = mutableListOf<NewNote>()
    override fun observeNotes() = flow { emit(answer()) }
    override suspend fun createNote(request: NewNote) = note(request.title).also { created += request }
}
```

A fake is a class, never an `object`; throw from it to test a failure. Interfaces everywhere are what make this possible.

## Repository: the real chain over MockEngine

```kotlin
private fun repository(engine: MockEngine, local: FakeNoteLocalDataSource = FakeNoteLocalDataSource()) =
    DefaultNoteRepository(DefaultNoteRemoteDataSource(ApiClient(apiHttpClient(target, HttpClient(engine)))), local)

@Test fun aServerErrorHasAReason() = runTest {
    val e = assertFailsWith<ReasonException> { repository(MockEngine { respond("", HttpStatusCode.InternalServerError) }).syncNotes() }
    assertEquals(ErrorReason.Server, e.reason)
}

@Test fun noAnswerIsOffline() = runTest {
    assertFailsWith<OfflineException> { repository(MockEngine { error("connection refused") }).syncNotes() }
}
```

- Where the wire matters (paths, JSON, status codes, the error mapping), test through the real client; a fake repository is for ViewModel tests.
- Check the cache is untouched when a sync fails. Plus: DTO → domain round trip, `fromWire` unknown value → `UNKNOWN`, an explicit `null` in JSON, an extra field the app doesn't know.

## Use case

Empty input, each boundary (midnight, month edge, week start, exact match), no match. Name tests as sentences.

## Where

`commonTest` by default. `jvmTest` only for code that needs JVM APIs. Robolectric (`androidHostTest`) for Android-only SDK wrappers.
