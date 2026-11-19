---
name: usecase
description: When to write a UseCase and its shape — a plain @Factory class <Verb><Thing>UseCase with operator fun invoke, written only when an action combines repositories or data sources, holds a rule worth naming and testing, or has a platform step; otherwise the ViewModel calls the repository directly. Lives in domain/ of the module that owns it and is unit-tested. Use when adding filtering, sorting, grouping, date maths, an action across several sources, or reviewing a use case that only forwards a call.
---

# UseCase

One action, named for what it does, called like a function.

## Shape

```kotlin
@Factory
class FilterNotesUseCase {
    operator fun invoke(notes: List<Note>, query: String): List<Note> =
        notes.filter { it.title.contains(query, ignoreCase = true) }
}

@Factory
class ShareNoteUseCase(private val notes: NoteRepository, private val links: LinkRepository, private val share: ShareSheet) {
    suspend operator fun invoke(id: NoteId) {
        val note = notes.note(id).first() ?: throw ReasonException(ErrorReason.NotFound)
        share.open(links.linkFor(note))
    }
}
```

## Write one only when

| The action… | Then |
|---|---|
| combines several repositories or data sources | a use case |
| holds a rule worth naming and testing (filter, group, date maths, validation) | a use case |
| has a platform step (share sheet, file picker, permission) behind an interface | a use case |
| reads or writes one area's data | no use case: the ViewModel calls the repository |
| holds state several screens observe | no use case: a repository's flow |

A use case that only forwards one call is an empty layer: don't write it.

## Rules

- **A plain class `<Verb><Thing>UseCase`**, `@Factory`, called as `operator fun invoke(...)`. Several functions only when they belong together (`offer()` and `answer()`). Never a top-level function or an `object`: it must be injectable and fakeable.
- **Calls only go down**: Screen → ViewModel → UseCase (when there is one) → Repository → data source. Never up, never past a layer, so each layer can be faked and tested alone.
- **Lives in `domain/`** of the owner: the feature, when one feature needs it; the service, when every consumer of the area does.
- **Errors propagate** as the repositories throw them (`OfflineException`, `ReasonException`); a rule that fails throws a `ReasonException` with its `ErrorReason`. No `Result<T>`.
- **Hands flows back, never collects them.** The ViewModel picks the scope.
- **No platform APIs in `commonMain`**: no `java.time`, `String.format`, `Locale`, `System.currentTimeMillis`. Use `kotlin.time.Clock` and `kotlinx.datetime`, with an explicit `TimeZone`.
- **Never logs sensitive data** it is handed.

## Test it

The layer with the best return. Every use case gets a `commonTest`: empty input, each boundary (midnight, month edge, exact match), no match, and a failing source.
