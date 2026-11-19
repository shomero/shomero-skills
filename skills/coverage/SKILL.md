---
name: coverage
description: Measure and defend unit-test coverage per layer with Kover in a KMP project — targets per layer, what to exclude (Compose UI, DI, generated code, SDK wrappers, dev tooling), running it on jvmTest, and a CI ratchet that fails the PR when a layer drops. Use when checking coverage, adding a module, ViewModel or service, or before merging.
---

# coverage

Kover measures the code unit tests can meaningfully cover. UI is covered by E2E flows; DI is config.

## Targets

| Layer | Packages | Target |
|---|---|---|
| Domain | `*.presentation` (ViewModels), `*.domain` (use cases) | 85–95% |
| Data | `*.data` (repositories, data sources, mappers), `*.network` | 60–75% |
| Excluded | `@Composable`, `@Preview`, Koin `@Module`, `*.di`, `org.koin.ksp.generated`, `*.generated.resources`, `*BuildKonfig`, `*ComposableSingletons*`, SDK wrappers, SQLDelight generated code, dev tooling | — |

Test **your** code that uses the network or DB (requests built, responses and errors mapped), not the SDK.

## Run it

```bash
./gradlew jvmTest          # NOT `test`: in KMP, commonTest runs via jvmTest
./gradlew koverHtmlReport  # build/reports/kover/html/index.html
./gradlew :services:notes:koverHtmlReport
```

## Wiring

- Root `build.gradle.kts`: `kover { reports { filters { excludes { annotatedBy(...); packages(...); classes(...) } } } }` and a `coveredModules` list.
- Each measured module applies `alias(libs.plugins.kover)` itself: membership is a per-module decision.

## Rules

1. New ViewModel, use case or domain helper → a `commonTest` in the same change.
2. New repository or mapper → success and failure of each `Result`, mapper round trip.
3. New module with logic → kover plugin + `coveredModules`. Pure UI or tooling → leave it out.
4. ViewModels don't live in `apps/`; put them in a feature module so they're measured.
5. Before merge: no layer below its target **or below its previous number**. Add tests; never lower a floor.

## The ratchet

Kover's `koverVerify` can't scope a floor to a package subset, so a script parses the Kover XML, prints the per-layer table, posts it on the PR, and exits non-zero under the floors (`COVERAGE_DOMAIN_FLOOR`, `COVERAGE_DATA_FLOOR`). Keep floors a few points under the real numbers and raise them as coverage climbs.

Needs the `ci-coverage-comment.sh` script from the example project.
