# Architecture reference

## The dependency rule

Dependencies point **inward**. Domain is the center and depends on nothing.

```
presentation ──▶ domain ◀── data
```

- **domain** imports only `dartz`, `equatable`, and other domain files. If a
  domain file imports `dio`, `flutter/*`, or anything from `data/` or
  `presentation/`, the layering is broken.
- **data** implements domain interfaces. Models extend domain entities. Data
  sources throw typed `Exception`s; the repository implementation catches them
  and returns `Either<Failure, T>`.
- **presentation** depends on domain use cases only — never on a data source or
  model directly.

## Layer responsibilities

| Layer | Contains | Knows about |
|---|---|---|
| domain/entities | plain immutable objects (`Equatable`) | nothing |
| domain/repositories | abstract repository interfaces | entities, `Failure` |
| domain/usecases | one action each, `call(params)` → `Either` | repository interface |
| data/models | entity + `fromJson`/`toJson` | its entity |
| data/datasources | remote (Dio) / local (storage); throw exceptions | models, dio, storage |
| data/repositories | implements domain interface; exceptions → failures | data sources, failures |
| presentation/bloc | events → states, calls use cases | use cases, entities |
| presentation/pages | widgets; render loading/success/error | its bloc |

## Why this shape

- **Testable**: domain and bloc are pure; you mock the repository/use case.
- **Swappable**: change Dio for another client, or secure storage for another
  backend, by editing only `data/` — domain and UI are untouched.
- **Predictable**: every feature looks the same, so a new file's home is obvious.

## Feature-add checklist

Build inner → outer so each file compiles:

- [ ] entity
- [ ] repository interface
- [ ] use case(s)
- [ ] model (`fromJson`/`toJson`, extends entity)
- [ ] remote data source (+ local if the feature caches)
- [ ] repository impl (try/catch → `Either`)
- [ ] register in `injection_container.dart`
- [ ] bloc + event + state
- [ ] page (+ widgets) with loading/error/empty handling
- [ ] route
- [ ] unit test (use case / repo), bloc test, widget test
- [ ] `flutter analyze` clean, `flutter test` green

## Adapting to an existing project

Before applying any template, **inspect the project**:

- Read `pubspec.yaml`. If it already uses a different state manager (Riverpod,
  Provider, Cubit-only), a different router, or a different DI approach, **follow
  that** instead of forcing this stack. Say so to the user and propose matching
  their convention.
- Match existing naming (`snake_case` files, `PascalCase` types), folder depth,
  and whether they split `event`/`state` into separate files or use `part of`.
- Reuse existing `core/` utilities (an existing `DioClient`, `Failure`, or
  `Result` type) rather than adding a parallel one.
- Add dependencies only if missing, with `flutter pub add`.

This skill's stack is the **default for greenfield or loosely-structured
projects**, not a mandate to rewrite a project that already has a consistent
architecture.
