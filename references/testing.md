# Testing reference

Dev dependencies: `flutter_test` (SDK), `bloc_test`, `mocktail`.

> **Rule:** never say a test passed unless you ran it and saw the result. Run
> `flutter test` (or `flutter test test/path/foo_test.dart`) and paste the real
> output. If you can't run it, say the test is written but unverified.

`mocktail` needs no code generation. Mirror `lib/` under `test/`
(`test/features/<feature>/...`).

---

## Unit test — use case (with mocktail)

```dart
import 'package:dartz/dartz.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

import 'package:myapp/core/error/failures.dart';
import 'package:myapp/features/profile/domain/entities/profile_entity.dart';
import 'package:myapp/features/profile/domain/repositories/profile_repository.dart';
import 'package:myapp/features/profile/domain/usecases/get_profile_usecase.dart';

class MockProfileRepository extends Mock implements ProfileRepository {}

void main() {
  late GetProfileUseCase useCase;
  late MockProfileRepository repository;

  setUp(() {
    repository = MockProfileRepository();
    useCase = GetProfileUseCase(repository);
  });

  const tProfile = ProfileEntity(id: '1', name: 'Ada', email: 'ada@x.com');

  test('returns profile from the repository on success', () async {
    when(() => repository.getProfile(any()))
        .thenAnswer((_) async => const Right(tProfile));

    final result = await useCase(const GetProfileParams(id: '1'));

    expect(result, const Right<Failure, ProfileEntity>(tProfile));
    verify(() => repository.getProfile('1')).called(1);
  });

  test('forwards a failure from the repository', () async {
    when(() => repository.getProfile(any()))
        .thenAnswer((_) async => const Left(ServerFailure('boom')));

    final result = await useCase(const GetProfileParams(id: '1'));

    expect(result, const Left<Failure, ProfileEntity>(ServerFailure('boom')));
  });
}
```

---

## Bloc test (with bloc_test)

```dart
import 'package:bloc_test/bloc_test.dart';
import 'package:dartz/dartz.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

import 'package:myapp/core/error/failures.dart';
import 'package:myapp/features/profile/domain/entities/profile_entity.dart';
import 'package:myapp/features/profile/domain/usecases/get_profile_usecase.dart';
import 'package:myapp/features/profile/presentation/bloc/profile_bloc.dart';
import 'package:myapp/features/profile/presentation/bloc/profile_event.dart';
import 'package:myapp/features/profile/presentation/bloc/profile_state.dart';

class MockGetProfileUseCase extends Mock implements GetProfileUseCase {}

class FakeGetProfileParams extends Fake implements GetProfileParams {}

void main() {
  late MockGetProfileUseCase getProfile;

  setUpAll(() => registerFallbackValue(FakeGetProfileParams()));
  setUp(() => getProfile = MockGetProfileUseCase());

  const tProfile = ProfileEntity(id: '1', name: 'Ada', email: 'ada@x.com');

  blocTest<ProfileBloc, ProfileState>(
    'emits [Loading, Loaded] when the use case succeeds',
    build: () {
      when(() => getProfile(any()))
          .thenAnswer((_) async => const Right(tProfile));
      return ProfileBloc(getProfile: getProfile);
    },
    act: (bloc) => bloc.add(const ProfileRequested('1')),
    expect: () => const [ProfileLoading(), ProfileLoaded(tProfile)],
  );

  blocTest<ProfileBloc, ProfileState>(
    'emits [Loading, Error] when the use case fails',
    build: () {
      when(() => getProfile(any()))
          .thenAnswer((_) async => const Left(ServerFailure('boom')));
      return ProfileBloc(getProfile: getProfile);
    },
    act: (bloc) => bloc.add(const ProfileRequested('1')),
    expect: () => const [ProfileLoading(), ProfileError('boom')],
  );
}
```

---

## Widget test

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

import 'package:myapp/features/profile/presentation/pages/profile_page.dart';

void main() {
  testWidgets('shows a loading indicator first', (tester) async {
    // If the page resolves its bloc from GetIt, register mocks in setUp,
    // or inject a bloc via a constructor/BlocProvider.value for testability.
    await tester.pumpWidget(
      const MaterialApp(home: ProfilePage(id: '1')),
    );
    expect(find.byType(CircularProgressIndicator), findsOneWidget);
  });
}
```

> Tip: pages that call `sl<Bloc>()` are hard to test. Prefer injecting the bloc
> (constructor param or `BlocProvider.value`) so tests can pass a mock, or
> register a fake in GetIt inside `setUp`.

---

## Running

```bash
flutter test                                   # all tests
flutter test test/features/profile             # one folder
flutter test --coverage                        # with coverage
```

Report the actual pass/fail counts. Fix failures before continuing.
