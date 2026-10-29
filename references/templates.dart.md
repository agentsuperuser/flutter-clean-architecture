# Feature templates reference

Templates for one feature. Replace the placeholders:

- `<Feature>` → PascalCase name (e.g. `Profile`)
- `<feature>` → snake_case name (e.g. `profile`)
- `<action>` → the use-case verb (e.g. `get`, `login`, `update`)

Create files in the order in `SKILL.md`. After each file, run `flutter analyze`
and fix before continuing.

---

## 1. domain/entities/&lt;feature&gt;_entity.dart

Pure, immutable, `Equatable`. No JSON, no Flutter.

```dart
import 'package:equatable/equatable.dart';

class ProfileEntity extends Equatable {
  const ProfileEntity({
    required this.id,
    required this.name,
    required this.email,
  });

  final String id;
  final String name;
  final String email;

  @override
  List<Object?> get props => [id, name, email];
}
```

---

## 2. domain/repositories/&lt;feature&gt;_repository.dart

Interface only. Returns `Either<Failure, T>`.

```dart
import 'package:dartz/dartz.dart';

import '../../../../core/error/failures.dart';
import '../entities/profile_entity.dart';

abstract class ProfileRepository {
  Future<Either<Failure, ProfileEntity>> getProfile(String id);
}
```

---

## 3. domain/usecases/&lt;action&gt;_usecase.dart

One action per use case. Use `NoParams` when there are no arguments; otherwise
make a small `Equatable` params class.

```dart
import 'package:dartz/dartz.dart';
import 'package:equatable/equatable.dart';

import '../../../../core/error/failures.dart';
import '../../../../core/usecases/usecase.dart';
import '../entities/profile_entity.dart';
import '../repositories/profile_repository.dart';

class GetProfileUseCase implements UseCase<ProfileEntity, GetProfileParams> {
  const GetProfileUseCase(this._repository);

  final ProfileRepository _repository;

  @override
  Future<Either<Failure, ProfileEntity>> call(GetProfileParams params) {
    return _repository.getProfile(params.id);
  }
}

class GetProfileParams extends Equatable {
  const GetProfileParams({required this.id});
  final String id;

  @override
  List<Object?> get props => [id];
}
```

---

## 4. data/models/&lt;feature&gt;_model.dart

Extends the entity; adds `fromJson`/`toJson`. Validate types defensively.

```dart
import '../../domain/entities/profile_entity.dart';

class ProfileModel extends ProfileEntity {
  const ProfileModel({
    required super.id,
    required super.name,
    required super.email,
  });

  factory ProfileModel.fromJson(Map<String, dynamic> json) {
    return ProfileModel(
      id: json['id'].toString(),
      name: (json['name'] ?? '') as String,
      email: (json['email'] ?? '') as String,
    );
  }

  Map<String, dynamic> toJson() => {
        'id': id,
        'name': name,
        'email': email,
      };
}
```

---

## 5. data/datasources/&lt;feature&gt;_remote_data_source.dart

Abstract + Dio impl. Throws typed exceptions (via `mapDioException`).

```dart
import 'package:dio/dio.dart';

import '../../../../core/error/exceptions.dart';
import '../../../../core/network/error_mapper.dart';
import '../models/profile_model.dart';

abstract class ProfileRemoteDataSource {
  Future<ProfileModel> getProfile(String id);
}

class ProfileRemoteDataSourceImpl implements ProfileRemoteDataSource {
  const ProfileRemoteDataSourceImpl(this._dio);

  final Dio _dio;

  @override
  Future<ProfileModel> getProfile(String id) async {
    try {
      final response = await _dio.get<Map<String, dynamic>>('/profiles/$id');
      final data = response.data;
      if (data == null) {
        throw const ServerException(message: 'Empty response');
      }
      return ProfileModel.fromJson(data);
    } on DioException catch (e) {
      throw mapDioException(e);
    }
  }
}
```

### Optional local data source (caching)

```dart
import 'dart:convert';

import 'package:shared_preferences/shared_preferences.dart';

import '../../../../core/error/exceptions.dart';
import '../models/profile_model.dart';

abstract class ProfileLocalDataSource {
  Future<ProfileModel?> getCachedProfile();
  Future<void> cacheProfile(ProfileModel profile);
}

class ProfileLocalDataSourceImpl implements ProfileLocalDataSource {
  const ProfileLocalDataSourceImpl(this._prefs);

  final SharedPreferences _prefs;
  static const _key = 'cached_profile';

  @override
  Future<ProfileModel?> getCachedProfile() async {
    final raw = _prefs.getString(_key);
    if (raw == null) return null;
    try {
      return ProfileModel.fromJson(
        jsonDecode(raw) as Map<String, dynamic>,
      );
    } catch (_) {
      throw const CacheException();
    }
  }

  @override
  Future<void> cacheProfile(ProfileModel profile) =>
      _prefs.setString(_key, jsonEncode(profile.toJson()));
}
```

> Do **not** cache tokens or secrets with SharedPreferences — use
> `SecureStorageService` (see `core.md`).

---

## 6. data/repositories/&lt;feature&gt;_repository_impl.dart

Catches exceptions and returns `Either`. This is the only place exceptions
become failures.

```dart
import 'package:dartz/dartz.dart';

import '../../../../core/error/exceptions.dart';
import '../../../../core/error/failures.dart';
import '../../domain/entities/profile_entity.dart';
import '../../domain/repositories/profile_repository.dart';
import '../datasources/profile_remote_data_source.dart';

class ProfileRepositoryImpl implements ProfileRepository {
  const ProfileRepositoryImpl({required ProfileRemoteDataSource remote})
      : _remote = remote;

  final ProfileRemoteDataSource _remote;

  @override
  Future<Either<Failure, ProfileEntity>> getProfile(String id) async {
    try {
      final profile = await _remote.getProfile(id);
      return Right(profile);
    } on NetworkException catch (e) {
      return Left(NetworkFailure(e.message));
    } on UnauthorizedException catch (e) {
      return Left(UnauthorizedFailure(e.message));
    } on ServerException catch (e) {
      return Left(ServerFailure(e.message, e.statusCode));
    } catch (_) {
      return const Left(ServerFailure('Unexpected error'));
    }
  }
}
```

---

## 7. presentation/bloc — event / state / bloc

**&lt;feature&gt;_event.dart**

```dart
import 'package:equatable/equatable.dart';

abstract class ProfileEvent extends Equatable {
  const ProfileEvent();

  @override
  List<Object?> get props => [];
}

class ProfileRequested extends ProfileEvent {
  const ProfileRequested(this.id);
  final String id;

  @override
  List<Object?> get props => [id];
}
```

**&lt;feature&gt;_state.dart**

```dart
import 'package:equatable/equatable.dart';

import '../../domain/entities/profile_entity.dart';

abstract class ProfileState extends Equatable {
  const ProfileState();

  @override
  List<Object?> get props => [];
}

class ProfileInitial extends ProfileState {
  const ProfileInitial();
}

class ProfileLoading extends ProfileState {
  const ProfileLoading();
}

class ProfileLoaded extends ProfileState {
  const ProfileLoaded(this.profile);
  final ProfileEntity profile;

  @override
  List<Object?> get props => [profile];
}

class ProfileError extends ProfileState {
  const ProfileError(this.message);
  final String message;

  @override
  List<Object?> get props => [message];
}
```

**&lt;feature&gt;_bloc.dart** — always emit Loading before the call, Loaded or
Error after. Map `Failure` → user-facing message via `fold`.

```dart
import 'package:flutter_bloc/flutter_bloc.dart';

import '../../domain/usecases/get_profile_usecase.dart';
import 'profile_event.dart';
import 'profile_state.dart';

class ProfileBloc extends Bloc<ProfileEvent, ProfileState> {
  ProfileBloc({required GetProfileUseCase getProfile})
      : _getProfile = getProfile,
        super(const ProfileInitial()) {
    on<ProfileRequested>(_onRequested);
  }

  final GetProfileUseCase _getProfile;

  Future<void> _onRequested(
    ProfileRequested event,
    Emitter<ProfileState> emit,
  ) async {
    emit(const ProfileLoading());
    final result = await _getProfile(GetProfileParams(id: event.id));
    result.fold(
      (failure) => emit(ProfileError(failure.message)),
      (profile) => emit(ProfileLoaded(profile)),
    );
  }
}
```

---

## 8. presentation/pages/&lt;feature&gt;_page.dart

Render every state. Use `BlocBuilder` for UI, `BlocListener` for one-off effects
(snackbars, navigation).

```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

import '../../../../injection_container.dart';
import '../bloc/profile_bloc.dart';
import '../bloc/profile_event.dart';
import '../bloc/profile_state.dart';

class ProfilePage extends StatelessWidget {
  const ProfilePage({super.key, required this.id});

  final String id;

  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => sl<ProfileBloc>()..add(ProfileRequested(id)),
      child: Scaffold(
        appBar: AppBar(title: const Text('Profile')),
        body: BlocConsumer<ProfileBloc, ProfileState>(
          listener: (context, state) {
            if (state is ProfileError) {
              ScaffoldMessenger.of(context).showSnackBar(
                SnackBar(content: Text(state.message)),
              );
            }
          },
          builder: (context, state) {
            if (state is ProfileLoading || state is ProfileInitial) {
              return const Center(child: CircularProgressIndicator());
            }
            if (state is ProfileError) {
              return _ErrorView(
                message: state.message,
                onRetry: () =>
                    context.read<ProfileBloc>().add(ProfileRequested(id)),
              );
            }
            if (state is ProfileLoaded) {
              final p = state.profile;
              return Padding(
                padding: const EdgeInsets.all(16),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(p.name, style: Theme.of(context).textTheme.titleLarge),
                    const SizedBox(height: 8),
                    Text(p.email),
                  ],
                ),
              );
            }
            return const SizedBox.shrink();
          },
        ),
      ),
    );
  }
}

class _ErrorView extends StatelessWidget {
  const _ErrorView({required this.message, required this.onRetry});

  final String message;
  final VoidCallback onRetry;

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          Text(message, textAlign: TextAlign.center),
          const SizedBox(height: 12),
          FilledButton(onPressed: onRetry, child: const Text('Retry')),
        ],
      ),
    );
  }
}
```

---

## Form validation pattern

For a form (e.g. login), validate with a `GlobalKey<FormState>` + `validator`s
before dispatching the submit event; show per-field errors and disable submit
while loading.

```dart
final _formKey = GlobalKey<FormState>();

TextFormField(
  controller: _emailController,
  keyboardType: TextInputType.emailAddress,
  decoration: const InputDecoration(labelText: 'Email'),
  validator: (v) {
    final value = v?.trim() ?? '';
    if (value.isEmpty) return 'Email is required';
    if (!value.contains('@')) return 'Enter a valid email';
    return null;
  },
),

// On submit:
if (_formKey.currentState?.validate() ?? false) {
  context.read<AuthBloc>().add(
        AuthLoginRequested(
          email: _emailController.text.trim(),
          password: _passwordController.text,
        ),
      );
}
```

Remember to `dispose()` controllers in a `StatefulWidget`.
