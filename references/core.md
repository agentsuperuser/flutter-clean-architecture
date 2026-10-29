# Core scaffolding reference

Copy-ready `core/` building blocks plus DI, router, and `main`. All are
feature-agnostic — set them up once per project. Adjust package import paths to
the project's package name (the `name:` in `pubspec.yaml`). These use relative
imports so they work under any package name.

---

## core/config/app_config.dart

Environment config read at compile time. Pass values with
`--dart-define-from-file=config/dev.json` (or individual `--dart-define`s).
**No secret ever lives in source** — the JSON config files are git-ignored.

```dart
/// Compile-time configuration. Override via:
///   flutter run --dart-define-from-file=config/dev.json
/// Never hardcode secrets here; supply them through --dart-define.
class AppConfig {
  const AppConfig._();

  static const String apiBaseUrl = String.fromEnvironment(
    'API_BASE_URL',
    defaultValue: 'https://api.example.com',
  );

  static const String environment = String.fromEnvironment(
    'ENV',
    defaultValue: 'development',
  );

  static const int connectTimeoutMs = int.fromEnvironment(
    'CONNECT_TIMEOUT_MS',
    defaultValue: 15000,
  );

  static const int receiveTimeoutMs = int.fromEnvironment(
    'RECEIVE_TIMEOUT_MS',
    defaultValue: 15000,
  );

  static const bool enableLogging = bool.fromEnvironment(
    'ENABLE_LOGGING',
    defaultValue: true,
  );

  static bool get isProduction => environment == 'production';
}
```

Example `config/dev.json` (git-ignore the whole `config/` dir, commit a
`config/dev.example.json` without secrets):

```json
{
  "API_BASE_URL": "https://dev.api.example.com",
  "ENV": "development",
  "ENABLE_LOGGING": true
}
```

---

## core/error/exceptions.dart

Thrown by data sources; caught in repository impls. Never surfaced to the UI.

```dart
/// Thrown inside the data layer. Repositories catch these and return a Failure.
class ServerException implements Exception {
  const ServerException({this.message = 'Server error', this.statusCode});
  final String message;
  final int? statusCode;
}

class NetworkException implements Exception {
  const NetworkException({this.message = 'No internet connection'});
  final String message;
}

class CacheException implements Exception {
  const CacheException({this.message = 'Cache error'});
  final String message;
}

class UnauthorizedException implements Exception {
  const UnauthorizedException({this.message = 'Unauthorized'});
  final String message;
}
```

---

## core/error/failures.dart

The left side of every `Either`. `props` is implemented (so two failures with
the same message compare equal — important for bloc state equality).

```dart
import 'package:equatable/equatable.dart';

abstract class Failure extends Equatable {
  const Failure(this.message);
  final String message;

  @override
  List<Object?> get props => [message];
}

class ServerFailure extends Failure {
  const ServerFailure([super.message = 'Server error', this.statusCode]);
  final int? statusCode;

  @override
  List<Object?> get props => [message, statusCode];
}

class NetworkFailure extends Failure {
  const NetworkFailure([super.message = 'No internet connection']);
}

class CacheFailure extends Failure {
  const CacheFailure([super.message = 'Cache error']);
}

class UnauthorizedFailure extends Failure {
  const UnauthorizedFailure([super.message = 'Unauthorized']);
}

class ValidationFailure extends Failure {
  const ValidationFailure([super.message = 'Validation error']);
}
```

---

## core/usecases/usecase.dart

```dart
import 'package:dartz/dartz.dart';
import 'package:equatable/equatable.dart';

import '../error/failures.dart';

/// Every use case implements this single-method contract.
/// Note: the return type param is `T`, not `Type` — `Type` is a dart core
/// type name and `flutter_lints` flags it (avoid_types_as_parameter_names).
abstract class UseCase<T, Params> {
  Future<Either<Failure, T>> call(Params params);
}

/// Params type for use cases that take no arguments.
class NoParams extends Equatable {
  const NoParams();

  @override
  List<Object?> get props => [];
}
```

---

## core/storage/secure_storage_service.dart

Tokens go here (Keychain / Keystore-backed), **never** in SharedPreferences.

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class SecureStorageService {
  SecureStorageService(this._storage);

  final FlutterSecureStorage _storage;

  static const _accessTokenKey = 'access_token';
  static const _refreshTokenKey = 'refresh_token';

  Future<void> saveAccessToken(String token) =>
      _storage.write(key: _accessTokenKey, value: token);

  Future<String?> readAccessToken() => _storage.read(key: _accessTokenKey);

  Future<void> saveRefreshToken(String token) =>
      _storage.write(key: _refreshTokenKey, value: token);

  Future<String?> readRefreshToken() => _storage.read(key: _refreshTokenKey);

  Future<bool> hasToken() async {
    final token = await readAccessToken();
    return token != null && token.isNotEmpty;
  }

  Future<void> clear() async {
    await _storage.delete(key: _accessTokenKey);
    await _storage.delete(key: _refreshTokenKey);
  }
}
```

---

## core/network/auth_interceptor.dart

```dart
import 'package:dio/dio.dart';

import '../storage/secure_storage_service.dart';

/// Attaches the bearer token to every request; clears it on a 401 so the
/// router can send the user back to login. Add refresh-token logic here.
class AuthInterceptor extends Interceptor {
  AuthInterceptor(this._storage);

  final SecureStorageService _storage;

  @override
  Future<void> onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    final token = await _storage.readAccessToken();
    if (token != null && token.isNotEmpty) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    handler.next(options);
  }

  @override
  Future<void> onError(
    DioException err,
    ErrorInterceptorHandler handler,
  ) async {
    if (err.response?.statusCode == 401) {
      await _storage.clear();
    }
    handler.next(err);
  }
}
```

---

## core/network/error_mapper.dart

```dart
import 'package:dio/dio.dart';

import '../error/exceptions.dart';

/// Maps a low-level DioException to one of our typed exceptions.
/// Call inside data sources; the repository then maps exception -> Failure.
Exception mapDioException(DioException e) {
  switch (e.type) {
    case DioExceptionType.connectionTimeout:
    case DioExceptionType.sendTimeout:
    case DioExceptionType.receiveTimeout:
    case DioExceptionType.connectionError:
      return const NetworkException();
    case DioExceptionType.badResponse:
      final status = e.response?.statusCode;
      if (status == 401) return const UnauthorizedException();
      final message = _messageFromResponse(e.response?.data) ?? 'Server error';
      return ServerException(message: message, statusCode: status);
    case DioExceptionType.cancel:
    case DioExceptionType.badCertificate:
    case DioExceptionType.unknown:
    default: // future-proof: newer dio versions may add enum cases
      return const ServerException(message: 'Something went wrong');
  }
}

String? _messageFromResponse(dynamic data) {
  if (data is Map<String, dynamic>) {
    final value = data['message'] ?? data['error'] ?? data['detail'];
    if (value is String) return value;
  }
  return null;
}
```

---

## core/network/dio_client.dart

```dart
import 'package:dio/dio.dart';

import '../config/app_config.dart';

/// Builds a configured Dio. Base URL + timeouts come from AppConfig (env).
class DioClient {
  DioClient({List<Interceptor> interceptors = const []}) {
    dio = Dio(
      BaseOptions(
        baseUrl: AppConfig.apiBaseUrl,
        connectTimeout: Duration(milliseconds: AppConfig.connectTimeoutMs),
        receiveTimeout: Duration(milliseconds: AppConfig.receiveTimeoutMs),
        headers: const {'Content-Type': 'application/json'},
      ),
    );
    dio.interceptors.addAll(interceptors);
    if (AppConfig.enableLogging && !AppConfig.isProduction) {
      dio.interceptors.add(
        LogInterceptor(requestBody: true, responseBody: true),
      );
    }
  }

  late final Dio dio;
}
```

---

## core/theme/app_theme.dart

```dart
import 'package:flutter/material.dart';

class AppTheme {
  const AppTheme._();

  static const Color _seed = Color(0xFF3F51B5);

  static ThemeData get light => ThemeData(
        useMaterial3: true,
        colorScheme: ColorScheme.fromSeed(seedColor: _seed),
        inputDecorationTheme: const InputDecorationTheme(
          border: OutlineInputBorder(),
        ),
      );

  static ThemeData get dark => ThemeData(
        useMaterial3: true,
        colorScheme: ColorScheme.fromSeed(
          seedColor: _seed,
          brightness: Brightness.dark,
        ),
        inputDecorationTheme: const InputDecorationTheme(
          border: OutlineInputBorder(),
        ),
      );
}
```

---

## injection_container.dart (GetIt)

Blocs are **factories** (fresh per screen); everything else is a **lazy
singleton**. Register one `_init<Feature>()` per feature.

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:get_it/get_it.dart';
import 'package:shared_preferences/shared_preferences.dart';

import 'core/network/auth_interceptor.dart';
import 'core/network/dio_client.dart';
import 'core/storage/secure_storage_service.dart';
// import feature files (bloc, use cases, repository, data sources) ...

final sl = GetIt.instance;

Future<void> init() async {
  // External (already-resolved instances use registerSingleton).
  final prefs = await SharedPreferences.getInstance();
  sl.registerSingleton<SharedPreferences>(prefs);
  sl.registerLazySingleton<FlutterSecureStorage>(
    () => const FlutterSecureStorage(),
  );

  // Core.
  sl.registerLazySingleton(() => SecureStorageService(sl()));
  sl.registerLazySingleton(() => AuthInterceptor(sl()));
  sl.registerLazySingleton(
    () => DioClient(interceptors: [sl<AuthInterceptor>()]),
  );

  // Features.
  _initAuth();
}

void _initAuth() {
  // Bloc (factory).
  // sl.registerFactory(() => AuthBloc(loginUseCase: sl(), ...));

  // Use cases (lazy singletons).
  // sl.registerLazySingleton(() => LoginUseCase(sl()));

  // Repository.
  // sl.registerLazySingleton<AuthRepository>(
  //   () => AuthRepositoryImpl(remote: sl(), local: sl()),
  // );

  // Data sources — remote takes the configured Dio.
  // sl.registerLazySingleton<AuthRemoteDataSource>(
  //   () => AuthRemoteDataSourceImpl(sl<DioClient>().dio),
  // );
  // sl.registerLazySingleton<AuthLocalDataSource>(
  //   () => AuthLocalDataSourceImpl(sl()),
  // );
}
```

---

## core/router/app_router.dart (go_router with auth redirect)

```dart
import 'dart:async';

import 'package:flutter/foundation.dart';
import 'package:go_router/go_router.dart';

// import your AuthBloc + states + pages ...

class AppRouter {
  AppRouter({required this.authBloc});

  final AuthBloc authBloc;

  late final GoRouter router = GoRouter(
    initialLocation: '/splash',
    refreshListenable: GoRouterRefreshStream(authBloc.stream),
    redirect: (context, state) {
      final authState = authBloc.state;
      final goingToLogin = state.matchedLocation == '/login';
      final goingToSplash = state.matchedLocation == '/splash';

      if (authState is AuthUnauthenticated || authState is AuthError) {
        return goingToLogin ? null : '/login';
      }
      if (authState is AuthAuthenticated && (goingToLogin || goingToSplash)) {
        return '/home';
      }
      return null;
    },
    routes: [
      GoRoute(path: '/splash', builder: (c, s) => const SplashPage()),
      GoRoute(path: '/login', builder: (c, s) => const LoginPage()),
      GoRoute(path: '/home', builder: (c, s) => const HomePage()),
    ],
  );
}

/// Bridges a bloc Stream to a Listenable so go_router re-evaluates redirects.
class GoRouterRefreshStream extends ChangeNotifier {
  GoRouterRefreshStream(Stream<dynamic> stream) {
    notifyListeners();
    _subscription =
        stream.asBroadcastStream().listen((_) => notifyListeners());
  }

  late final StreamSubscription<dynamic> _subscription;

  @override
  void dispose() {
    _subscription.cancel();
    super.dispose();
  }
}
```

---

## main.dart

```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

import 'core/router/app_router.dart';
import 'core/theme/app_theme.dart';
import 'injection_container.dart' as di;
// import AuthBloc + AuthCheckRequested ...

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await di.init();
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    final authBloc = di.sl<AuthBloc>()..add(const AuthCheckRequested());
    final appRouter = AppRouter(authBloc: authBloc);

    return BlocProvider.value(
      value: authBloc,
      child: MaterialApp.router(
        title: 'App',
        debugShowCheckedModeBanner: false,
        theme: AppTheme.light,
        darkTheme: AppTheme.dark,
        routerConfig: appRouter.router,
      ),
    );
  }
}
```
