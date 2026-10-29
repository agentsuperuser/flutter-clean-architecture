---
name: flutter-clean-architecture
description: >
  Build production-ready Flutter apps with feature-first Clean Architecture
  (data/domain/presentation), BLoC state management, Dio networking, GetIt
  dependency injection, go_router, dartz Either error handling, secure token
  storage, and environment-based config. Use when creating or extending a
  Flutter app or feature, wiring authentication or API calls, setting up DI,
  routing, themes, forms, or tests, or when the user mentions clean
  architecture, BLoC, Dio, GetIt, or production Flutter structure.
---

# Flutter Clean Architecture

A reusable workflow for building and extending **production-ready Flutter apps**
with feature-first Clean Architecture. Works for any Flutter project — a fresh
app or an existing one.

## The architecture

```
Presentation (UI + BLoC)  ──depends on──▶  Domain (entities, repo interfaces, use cases)  ◀──depends on──  Data (models, data sources, repo impl)
                                                      ▲
                                        Domain depends on NOTHING outside itself
```

- **domain/** — pure Dart. Entities (`Equatable`), repository *interfaces*, and
  use cases. No Flutter, no Dio, no imports from data/ or presentation/.
- **data/** — models (extend entities, add `fromJson`/`toJson`), data sources
  (remote via Dio, local via secure storage / shared_preferences), and the
  repository *implementation* that maps exceptions → `Failure` and returns
  `Either<Failure, T>`.
- **presentation/** — BLoC (events/states) and widgets/pages. Never touches a
  data source directly — only use cases.

Shared, cross-feature code lives in **core/** (errors, network, config, DI,
router, theme, base use case).

### Directory layout

```
lib/
├── core/
│   ├── config/app_config.dart          # env via String.fromEnvironment
│   ├── error/{failures.dart,exceptions.dart}
│   ├── network/{dio_client.dart,auth_interceptor.dart,error_mapper.dart}
│   ├── storage/secure_storage_service.dart
│   ├── usecases/usecase.dart
│   ├── theme/app_theme.dart
│   └── router/app_router.dart
├── features/<feature>/
│   ├── data/{datasources,models,repositories}/
│   ├── domain/{entities,repositories,usecases}/
│   └── presentation/{bloc,pages,widgets}/
├── injection_container.dart            # GetIt: sl + init()
└── main.dart
```

## Order to add a feature (always this order)

1. `domain/entities/<feature>_entity.dart`
2. `domain/repositories/<feature>_repository.dart` (interface)
3. `domain/usecases/<action>_usecase.dart`
4. `data/models/<feature>_model.dart`
5. `data/datasources/<feature>_remote_data_source.dart` (+ local if needed)
6. `data/repositories/<feature>_repository_impl.dart`
7. register everything in `injection_container.dart`
8. `presentation/bloc/<feature>_{bloc,event,state}.dart`
9. `presentation/pages/<feature>_page.dart` (+ widgets)
10. add the route in `core/router/app_router.dart`
11. tests (unit + bloc, and a widget test for the page)

Build the inner layers before the outer ones so each file compiles against code
that already exists.

## Non-negotiable workflow

Follow these on every change (they override speed):

1. **Inspect first.** Read the existing project — `pubspec.yaml`, folder layout,
   naming, state-management and DI conventions — before adding anything. Match
   what is already there; do not impose this structure on a project that already
   has a different, consistent one.
2. **Follow the project's established architecture**, not a generic ideal.
3. **One file at a time.** Create or edit a single file, then stop.
4. **Explain each file briefly** — one or two lines on what it does and why.
5. **Run `flutter analyze` after each step** when a terminal is available.
6. **Fix every error and warning before moving on.** Never stack changes on top
   of a broken analyze.
7. **Never claim tests passed unless you actually ran them** and saw the output.
   Paste the real result. If you could not run them, say so.
8. **No unnecessary dependencies and no unrelated edits.** Add a package only
   when the task needs it; justify it in one line.
9. **Never hardcode credentials, passwords, API keys, or tokens.** Use
   environment config (`--dart-define-from-file`) and store tokens in
   `flutter_secure_storage`. Keep secrets out of source control.

Also always: handle **loading, success, and error** states in the UI, and
**validate forms** before submitting.

## Dependencies this skill uses

Runtime: `flutter_bloc`, `equatable`, `dartz`, `get_it`, `go_router`, `dio`,
`flutter_secure_storage`, `shared_preferences`.
Dev: `bloc_test`, `mocktail`, `flutter_lints`.

Add with `flutter pub add <pkg>` (and `flutter pub add dev:<pkg>` for dev deps)
so versions resolve against the project's SDK. Don't pin versions by hand unless
asked. Prefer hand-written `fromJson`/`toJson` + `equatable`; reach for
`freezed`/`json_serializable` only if the project already uses codegen.

## Reference files (read the one you need)

- **references/architecture.md** — layers, the dependency rule, feature-add
  checklist, and how to adapt to an existing project.
- **references/core.md** — copy-ready `core/` scaffolding: failures/exceptions,
  base `UseCase`, Dio client + auth interceptor + error mapper, secure storage,
  `AppConfig` env, GetIt `init()`, go_router with auth redirect, theme, `main`.
- **references/templates.dart.md** — per-layer templates for a generic
  `<feature>` (entity → model → data sources → repo impl → use case → bloc →
  page) with loading/error/validation wired in.
- **references/testing.md** — unit (mocktail), bloc (`bloc_test`), and widget
  test patterns, plus how to run them and report real results.
- **references/checklist.md** — the production & security checklist and the
  analyze→fix loop to run before calling a feature done.
