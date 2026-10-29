# Production & security checklist

Run through this before calling a feature (or the app) done.

## The analyze → fix loop (after every file)

```bash
flutter analyze        # must end with "No issues found!"
```

- Fix every error **and** warning before the next file. Don't let issues pile up.
- If adding a dependency: `flutter pub get`, then analyze again.
- Before a PR: `dart format .` then `flutter analyze` then `flutter test`.

## Security (hard rules)

- [ ] **No hardcoded credentials, passwords, API keys, or tokens** anywhere in
      source. Supply config via `--dart-define-from-file`.
- [ ] Tokens stored only in `flutter_secure_storage`, never SharedPreferences.
- [ ] Secret config files (`config/*.json`, `.env`) are in `.gitignore`; commit
      only a `*.example` without real values.
- [ ] Tokens never logged. `LogInterceptor` runs only when
      `enableLogging && !isProduction`.
- [ ] 401 handling clears the token and routes to login.
- [ ] No secrets echoed into terminal output, error messages, or analytics.
- [ ] Input is validated before it reaches the network.

## Architecture

- [ ] Domain imports nothing from `data/`, `presentation/`, Flutter, or Dio.
- [ ] UI talks to use cases, never directly to a data source or model.
- [ ] Exceptions are caught in the repository impl and converted to `Failure`;
      they never reach the bloc or UI.
- [ ] Models extend entities and own `fromJson`/`toJson`.
- [ ] One use case = one action.

## State & UI

- [ ] Loading, success, and error states are all rendered.
- [ ] Errors offer a retry where it makes sense.
- [ ] Forms validate before submit; submit is disabled while loading.
- [ ] Equatable `props` include every field (so bloc states compare correctly).
- [ ] `const` constructors used where possible.
- [ ] Controllers / focus nodes / subscriptions disposed.
- [ ] No `BuildContext` used across an `await` without a `context.mounted` guard.

## DI

- [ ] Blocs registered as `registerFactory`; everything else lazy singletons.
- [ ] Async externals (SharedPreferences) awaited in `init()`.
- [ ] Interfaces registered by their abstract type
      (`registerLazySingleton<AuthRepository>(...)`).

## Tests

- [ ] At least one unit test (use case or repo) and one bloc test per feature.
- [ ] Tests were actually run; real output reported.
- [ ] `flutter test` green before done.

## Dependencies

- [ ] Only packages the task needs; each new one justified in one line.
- [ ] No unrelated edits bundled into the change.
