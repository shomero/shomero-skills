---
name: repository
description: What a Repository is — an <Area>Repository interface with a Default<Area>Repository bound by @Single(binds = ...) in services/, returning values and Flows and letting OfflineException and ReasonException propagate, the local database as the source of truth for cached areas, the network touched only by an explicit sync*, and writes that go remote-first then into the cache. Use when adding or reviewing a repository or a services/ module.
---

# Repository

The feature-facing API for one domain area. Data access only; it decides nothing about UI.

## Shape

```kotlin
interface NoteRepository {
    fun observeNotes(): Flow<List<Note>>              // the cache, updated by sync and writes
    fun note(id: NoteId): Flow<Note?>
    suspend fun syncNotes()                           // the one network read
    suspend fun createNote(request: NewNote): Note
}

@Single(binds = [NoteRepository::class])
class DefaultNoteRepository(
    private val remote: NoteRemoteDataSource,
    private val local: NoteLocalDataSource,
) : NoteRepository {
    override fun observeNotes() = local.observeAll()
    override fun note(id: NoteId) = local.observe(id)
    override suspend fun syncNotes() = local.replaceAll(remote.notes())
    override suspend fun createNote(request: NewNote) = remote.create(request).also { local.upsert(it) }
}
```

An area with no cache returns a cold flow a screen's `load()` collects, so it runs again on each `reload()`:

```kotlin
override fun health(): Flow<Health> = flow { emit(remote.health()) }
```

## Rules

- **Errors are exceptions.** The API client throws `OfflineException` (no answer) or `ReasonException(ErrorReason)`; the repository lets them propagate. No `Result<T>`, no `.catch { emit(null) }` or `emptyList()`: that shows "nothing here" when the network is down. The ViewModel maps them with `failed(e)`.
- **The cache is the source of truth** for an area screens observe. The screen renders last-known data instantly, works offline, and every observer of a row updates together. Returning a network response straight to the UI is how "I edited it, went back, and it didn't change" bugs happen.
- **Writes: remote first, then upsert the result** (or re-fetch into the cache). A mutation that skips the cache leaves other screens stale.
- **Optimistic writes** apply locally first and roll back on failure, only when the UX needs it.
- **Interface + `Default<Interface>`**, bound with `@Single(binds = [...])`, or `@Factory(binds = [...])` when it's built on a client that changes with sign-in (or it keeps the old session). Never an `object`.
- **Main-safe.** Local data sources switch to their injected dispatcher; Ktor calls already are. `Dispatchers.IO` doesn't exist on native.
- **Not every repository needs a cache**: a one-shot action ("send feedback") is remote-only. Sensitive data the project forbids persisting is fetched on demand and held in memory.
- **Mirror the backend surface.** A new service implements the whole matching API group, so the client is complete before every screen exists.
- **Cross-service reads** go through the other service's public repository or `<Area>LocalDataSource` interface, never its tables.
- **On a server, the database is the repository's implementation** (`Pg<Area>Repository`, with a `Fake<Area>Repository` for tests), not a data source: it's the record itself. A rule the repository must check inside its own transaction stays there, so the read, the check and the write stay atomic.

## Checklist

- [ ] `<Area>Repository` interface, `Default<Area>Repository` bound to it
- [ ] Returns values or `Flow`s; no `Result`, no swallowed failure
- [ ] `observe*` reads only the local database
- [ ] Mutations refresh the cache on success
- [ ] Tests: the real chain over `MockEngine` for success, `ReasonException` and `OfflineException`; cache untouched when a sync fails (see `testing-kmp`)
