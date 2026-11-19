---
name: data-source
description: What a DataSource is — the only code that talks to the wire or the database. An <Area>RemoteDataSource calls the injected API client, which turns failures into OfflineException or ReasonException, with @Serializable DTOs and toDomain/toDto mappers; an <Area>LocalDataSource wraps SQLDelight queries or the settings Store. Interfaces with Default implementations, both returning domain types. Use when adding a data source, a DTO, an enum from the wire, or an endpoint.
---

# DataSource

## Remote

```kotlin
interface NoteRemoteDataSource {
    suspend fun notes(): List<Note>
    suspend fun create(request: NewNote): Note
}

@Single(binds = [NoteRemoteDataSource::class])
class DefaultNoteRemoteDataSource(private val api: ApiClient) : NoteRemoteDataSource {
    override suspend fun notes() = api.request { get(NOTES).body<NotesResponseDto>() }.notes.map(NoteDto::toDomain)
    override suspend fun create(request: NewNote) = api.request { post(NOTES) { setBody(request.toDto()) }.body<NoteDto>() }.toDomain()
}

private const val NOTES = "notes"
```

- **Inject the app's single API client.** It owns auth headers and token refresh; never build an `HttpClient` here.
- **Every call goes through the client's `request { }`**, which turns failures into `OfflineException` or `ReasonException` (see `feature-network`). Catch nothing here: let them propagate.
- **`@Factory(binds = [...])`** instead of `@Single` when the client changes with sign-in, so it never keeps the old session.
- **Routes are private constants.** No sensitive value in a path or query: opaque ids only, a URL ends up in server and proxy logs.

## DTOs and mappers

- `@Serializable` wire DTOs under `data/sources/remote/dto/`, `@SerialName("snake_case")`. Domain models carry no serialization annotations; convert at the boundary with `toDomain()` / `toDto()`.
- **Explicit JSON nulls** fail a non-nullable field with a default: make it nullable and default in the mapper (or `coerceInputValues`).
- **An id has no default.** A missing id should fail decoding, not decode to `""` and crash a `LazyColumn` key later.
- Partial-update bodies use `Json { explicitNulls = false }` so unset fields are omitted.

## Enums from the wire

```kotlin
enum class Tier(val wire: String) { FREE("free"), PRO("pro"), UNKNOWN("");
    companion object { fun fromWire(v: String?) = entries.firstOrNull { it.wire == v } ?: UNKNOWN } }
```

Always `?: UNKNOWN`, never a real value; and never write `UNKNOWN` back on create/update.

## Local

```kotlin
@Single(binds = [NoteLocalDataSource::class])
class DefaultNoteLocalDataSource(db: DatabaseProvider, private val dispatcher: CoroutineDispatcher) : NoteLocalDataSource {
    private val q = db.database.noteQueries
    override fun observeAll() = q.selectAll().asFlow().mapToList(dispatcher).map { it.map(::toDomain) }
    override suspend fun replaceAll(items: List<Note>) = withContext(dispatcher) {
        q.transaction { q.deleteAll(); items.forEach { q.insert(it.toRow()) } }
    }
}
```

- Reads observe; writes go in a `transaction`. Adding a table: see `sqldelight-table`.
- Small key-value values (a flag, the theme) are a local data source over the settings `Store` (see `feature-settings`).
- **Local is anything offline or on the device** (settings, saved sign-ins, files); remote is anything that goes over the network.
- **Platform library interfaces keep their library's names** (`Store`, `Connectivity`, `Push`): a data source wraps them or has them injected, it doesn't rename them.
