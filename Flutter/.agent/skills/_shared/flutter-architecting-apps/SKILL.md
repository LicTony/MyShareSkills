---
name: flutter-architecting-apps
description: Architects a Flutter application using the recommended layered approach (UI, Logic, Data) with modern state management. Use when structuring a new project or refactoring for scalability.
metadata:
  model: models/gemini-3.1-pro-preview
  last_modified: Thu, 2 Apr 2026 00:00:00 GMT

---
# Architecting Flutter Applications

> **Version:** 1.0.0 — 2026-04-02

## Contents
- [Core Architectural Principles](#core-architectural-principles)
- [Structuring the Layers](#structuring-the-layers)
- [Implementing the Data Layer](#implementing-the-data-layer)
- [State Management Options](#state-management-options)
- [Dependency Injection](#dependency-injection)
- [Feature Implementation Workflow](#feature-implementation-workflow)
- [Testing Strategy](#testing-strategy)
- [Examples](#examples)

## Core Architectural Principles

Design Flutter applications to scale by strictly adhering to the following principles:

*   **Enforce Separation of Concerns:** Decouple UI rendering from business logic and data fetching. Organize the codebase into distinct layers (UI, Logic, Data) and further separate by feature within those layers.
*   **Maintain a Single Source of Truth (SSOT):** Centralize application state and data in the Data layer. Ensure the SSOT is the only component authorized to mutate its respective data.
*   **Implement Unidirectional Data Flow (UDF):** Flow state downwards from the Data layer to the UI layer. Flow events upwards from the UI layer to the Data layer.
*   **Treat UI as a Function of State:** Drive the UI entirely via immutable state objects. Rebuild widgets reactively when the underlying state changes.
*   **Prefer Composition over Inheritance:** Build UIs by composing small, reusable widgets rather than extending base classes.

## Structuring the Layers

Separate the application into 2 to 3 distinct layers depending on complexity. Restrict communication so that a layer only interacts with the layer directly adjacent to it.

### 1. UI Layer (Presentation)
*   **Views (Widgets):** Build reusable, lean widgets. Strip all business and data-fetching logic from the widget tree. Restrict widget logic to UI-specific concerns (e.g., animations, routing, layout constraints).
*   **ViewModels / Notifiers:** Manage the UI state. Consume domain models from the Data/Logic layers and transform them into presentation-friendly formats. Expose state to the Views and handle user interaction events.
*   **State Holders:** Use modern state management solutions (Riverpod, Signals, or ChangeNotifier) based on project complexity.

### 2. Logic Layer (Domain) - *Conditional*
*   **If the application requires complex client-side business logic:** Implement a Logic layer containing Use Cases or Interactors. Use this layer to orchestrate interactions between multiple repositories before passing data to the UI layer.
*   **If the application is a standard CRUD app:** Omit this layer. Allow ViewModels to interact directly with Repositories.

### 3. Data Layer (Model)
*   **Responsibilities:** Act as the SSOT for all application data. Handle business data, external API consumption, event processing, and data synchronization.
*   **Components:** Divide the Data layer strictly into **Repositories** and **Services**.

## Implementing the Data Layer

### Services
*   **Role:** Wrap external APIs (HTTP servers, local databases, platform plugins).
*   **Implementation:** Write Services as stateless Dart classes. Do not store application state here.
*   **Mapping:** Create exactly one Service class per external data source.
*   **Recommended Tools:** Use `dio` for HTTP clients, `retrofit` for type-safe API definitions, `hive` or `isar` for local storage.

### Repositories
*   **Role:** Act as the SSOT for domain data.
*   **Implementation:** Consume raw data from Services. Handle caching, offline synchronization, and retry logic.
*   **Transformation:** Transform raw API/Service data into clean Domain Models formatted for consumption by ViewModels.
*   **Error Handling:** Wrap external exceptions in domain-specific error types (sealed classes).

## State Management Options

Choose the state management solution based on project requirements:

### Option 1: Riverpod (Recommended for New Projects)
**Use when:** Building scalable apps with complex state dependencies and testing requirements.

```dart
// Provider definition
@riverpod
Future<User> user(UserRef ref, String userId) async {
  final repository = ref.watch(userRepositoryProvider);
  return repository.getUser(userId);
}

// Usage in widget
class UserProfileView extends ConsumerWidget {
  final String userId;
  
  const UserProfileView({required this.userId});
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final userAsync = ref.watch(userProvider(userId));
    
    return userAsync.when(
      data: (user) => Text('Hello, ${user.name}'),
      loading: () => const CircularProgressIndicator(),
      error: (error, _) => Text('Error: $error'),
    );
  }
}
```

// Option 2: Signals (Emerging Standard)
**Use when:** Want fine-grained reactivity with minimal boilerplate.

```dart
// Signal Controller (Encapsulated State)
class UserSignalsController {
  final UserRepository _repository;
  
  UserSignalsController(this._repository);

  final user = signal<User?>(null);
  final isLoading = signal(false);
  final error = signal<String?>(null);

  void loadUser(String userId) async {
    isLoading.value = true;
    error.value = null;

    try {
      user.value = await _repository.getUser(userId);
    } catch (e) {
      error.value = e.toString();
    } finally {
      isLoading.value = false;
    }
  }
}

// Consumption in widget (using signals_flutter)
class UserProfileSignalsView extends StatelessWidget {
  final UserSignalsController controller;
  
  const UserProfileSignalsView({required this.controller});

  @override
  Widget build(BuildContext context) {
    return Watch((context) {
      if (controller.isLoading.value) return const CircularProgressIndicator();
      if (controller.error.value != null) return Text('Error: ${controller.error.value}');
      if (controller.user.value == null) return const Text('No user');

      return Text('Hello, ${controller.user.value!.name}');
    });
  }
}
```

### Option 3: ChangeNotifier (Legacy/Migration)
**Use when:** Maintaining existing codebases or for simple prototypes.

```dart
class UserViewModel extends ChangeNotifier {
  final UserRepository _userRepository;
  User? _user;
  bool _isLoading = false;
  String? _error;

  User? get user => _user;
  bool get isLoading => _isLoading;
  String? get error => _error;

  UserViewModel(this._userRepository);

  Future<void> loadUser(String userId) async {
    _isLoading = true;
    _error = null;
    notifyListeners();

    try {
      _user = await _userRepository.getUser(userId);
    } catch (e) {
      _error = e.toString();
    } finally {
      _isLoading = false;
      notifyListeners();
    }
  }
}
```

## Dependency Injection

Use a DI solution to manage dependencies across layers:

### GetIt + Injectable (Recommended)
```dart
// Register dependencies
@injectable
class UserViewModel {
  final UserRepository _repository;
  
  UserViewModel(this._repository);
}

@module
abstract class RegisterModule {
  @preResolve
  @singleton
  Future<Dio> get dio async => Dio(BaseOptions(baseUrl: '...'));
  
  @lazySingleton
  UserApiService get userApiService => UserApiService(dio);
  
  @lazySingleton
  UserRepository get userRepository => UserRepository(userApiService);
}

// Initialize
await configureInjection();
final viewModel = getIt<UserViewModel>();
```

## Feature Implementation Workflow

Follow this sequential workflow when adding a new feature to the application.

**Task Progress:**
- [ ] **Step 1: Define Domain Models.** Create immutable Dart classes using `freezed` for the core data structures.
- [ ] **Step 2: Define Error Types.** Create sealed classes for domain-specific errors.
- [ ] **Step 3: Implement Services.** Create stateless Service classes with `retrofit` (for APIs) or direct SDK usage.
- [ ] **Step 4: Implement Repositories.** Create Repository classes that consume Services, handle caching, and return Domain Models.
- [ ] **Step 5: Setup Dependency Injection.** Register Services and Repositories in GetIt/Injectable.
- [ ] **Step 6: Implement State Management.** Create Providers (Riverpod) or ViewModels that consume Repositories.
- [ ] **Step 7: Implement Views.** Create Flutter Widgets that bind to the state and trigger actions.
- [ ] **Step 8: Run Validator.** Execute unit tests for Services, Repositories, and ViewModels. Execute widget tests for Views.
    *   *Feedback Loop:* Review test failures -> Fix logic/mocking errors -> Re-run tests until passing.

## Testing Strategy

### Unit Tests (Data & Logic Layers)
```dart
// Test Repository with mocked Service
void main() {
  late MockUserApiService mockService;
  late UserRepository repository;
  
  setUp(() {
    mockService = MockUserApiService();
    repository = UserRepository(mockService);
  });
  
  test('getUser returns cached user on second call', () async {
    when(() => mockService.fetchUserRaw('1'))
      .thenAnswer((_) async => {'id': '1', 'name': 'John'});
    
    final user1 = await repository.getUser('1');
    final user2 = await repository.getUser('1');
    
    verify(() => mockService.fetchUserRaw('1')).called(1);
    expect(user1.id, equals(user2.id));
  });
}
```

### Widget Tests (UI Layer)
```dart
// Test Riverpod integration with Widget
void main() {
  testWidgets('UserProfileView shows loading state', (tester) async {
    await tester.pumpWidget(
      const ProviderScope(
        child: MaterialApp(
          home: UserProfileView(userId: '1'),
        ),
      ),
    );
    
    expect(find.byType(CircularProgressIndicator), findsOneWidget);
  });
}
```

### Golden Tests (Visual Regression)
```dart
// Test UI appearance
testWidgets('UserProfileView matches golden', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      overrides: [
        userProvider('1').overrideWith((ref) => testUser),
      ],
      child: const MaterialApp(
        home: UserProfileView(userId: '1'),
      ),
    ),
  );
  
  await expectLater(
    find.byType(UserProfileView),
    matchesGoldenFile('goldens/user_profile_view.png'),
  );
});
```

## Examples

### Domain Model with Freezed

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user.freezed.dart';
part 'user.g.dart';

@freezed
class User with _$User {
  const factory User({
    required String id,
    required String name,
    required String email,
    @JsonKey(name: 'created_at') required DateTime createdAt,
  }) = _User;
  
  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
}
```

### Error Handling with Sealed Classes

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user_errors.freezed.dart';

@freezed
class UserError with _$UserError {
  const factory UserError.notFound(String userId) = UserNotFound;
  const factory UserError.networkFailure(String message) = UserNetworkFailure;
  const factory UserError.cacheCorrupted() = UserCacheCorrupted;
  const factory UserError.unauthorized() = UserUnauthorized;
}
```

### Repository with Error Mapping

```dart
class UserRepository {
  final UserApiService _apiService;
  final CacheService _cacheService;
  User? _cachedUser;

  UserRepository(this._apiService, this._cacheService);

  Future<User> getUser(String userId) async {
    // Check cache first
    if (_cachedUser != null && _cachedUser!.id == userId) {
      return _cachedUser!;
    }

    try {
      final rawData = await _apiService.fetchUserRaw(userId);
      final user = User.fromJson(rawData);
      
      // Cache successful response
      await _cacheService.write('user_$userId', rawData);
      _cachedUser = user;
      
      return user;
    } on DioException catch (e) {
      // Try cache fallback on network error
      final cached = await _cacheService.read('user_$userId');
      if (cached != null) {
        return User.fromJson(cached);
      }
      
      if (e.response?.statusCode == 404) {
        throw UserError.notFound(userId);
      }
      throw UserError.networkFailure(e.message);
    }
  }
}
```

### Riverpod Provider Setup

```dart
// providers/user_providers.dart
@riverpod
UserRepository userRepository(UserRepositoryRef ref) {
  return UserRepository(
    ref.watch(userApiServiceProvider),
    ref.watch(cacheServiceProvider),
  );
}

@riverpod
Future<User> user(UserRef ref, String userId) async {
  final repository = ref.watch(userRepositoryProvider);
  return repository.getUser(userId);
}

@riverpod
class UserNotifier extends _$UserNotifier {
  @override
  Future<User?> build() async => null;
  
  Future<void> loadUser(String userId) async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() => 
      ref.read(userRepositoryProvider).getUser(userId)
    );
  }
  
  Future<void> updateUser(User user) async {
    final previousState = state;
    // Optimistic update
    state = AsyncValue.data(user);
    
    try {
      await ref.read(userRepositoryProvider).updateUser(user);
    } catch (e) {
      // Rollback on failure
      state = previousState;
      rethrow;
    }
  }
}
```

### View with Riverpod

```dart
class UserProfileView extends ConsumerWidget {
  final String userId;
  
  const UserProfileView({required this.userId});
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final userAsync = ref.watch(userProvider(userId));
    
    return Scaffold(
      appBar: AppBar(title: const Text('User Profile')),
      body: userAsync.when(
        data: (user) {
          if (user == null) {
            return const Center(child: Text('No user data'));
          }
          return Padding(
            padding: const EdgeInsets.all(16),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(user.name, style: Theme.of(context).textTheme.headlineSmall),
                const SizedBox(height: 8),
                Text(user.email, style: Theme.of(context).textTheme.bodyMedium),
              ],
            ),
          );
        },
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (error, stack) => Center(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              Icon(Icons.error, color: Theme.of(context).colorScheme.error),
              const SizedBox(height: 8),
              Text('Error: $error'),
            ],
          ),
        ),
      ),
    );
  }
}
```

### View with ChangeNotifier (Legacy)

```dart
class UserProfileView extends StatelessWidget {
  final UserViewModel viewModel;
  
  const UserProfileView({required this.viewModel});
  
  @override
  Widget build(BuildContext context) {
    return ListenableBuilder(
      listenable: viewModel,
      builder: (context, child) {
        if (viewModel.isLoading) {
          return const Center(child: CircularProgressIndicator());
        }
        
        if (viewModel.error != null) {
          return Center(child: Text('Error: ${viewModel.error}'));
        }
        
        if (viewModel.user == null) {
          return const Center(child: Text('No user data'));
        }
        
        return Padding(
          padding: const EdgeInsets.all(16),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(viewModel.user!.name, 
                   style: Theme.of(context).textTheme.headlineSmall),
              const SizedBox(height: 8),
              Text(viewModel.user!.email, 
                   style: Theme.of(context).textTheme.bodyMedium),
            ],
          ),
        );
      },
    );
  }
}
```

## Recommended Packages (2026)

| Category | Package | Purpose |
|----------|---------|---------|
| **State Management** | `flutter_riverpod` | Reactive state management |
| | `signals_flutter` | Fine-grained reactivity (Watch) |
| | `signals` | Core signal primitives |
| **Code Generation** | `freezed` | Immutable models with copyWith |
| | `json_serializable` | JSON serialization |
| **Networking** | `dio` | HTTP client |
| | `retrofit` | Type-safe API definitions |
| **Local Storage** | `hive` | Fast NoSQL database |
| | `isar` | Reactive local database |
| | `shared_preferences` | Simple key-value storage |
| **Dependency Injection** | `get_it` | Service locator |
| | `injectable` | Code generation for GetIt |
| **Testing** | `mockito` | Mock generation |
| | `mocktail` | Modern mocking (null-safe) |
| | `golden_toolkit` | Golden file testing |
