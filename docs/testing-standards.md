# Testing Standards

**Last updated:** 2026-05-18

## Testing Framework

- **Local Unit Tests:** JUnit 4 (`junit:4.12`) — runs on JVM, no Android device needed
- **Instrumented Tests:** AndroidX Test Runner (`androidx.test.runner.AndroidJUnitRunner`) — runs on emulator or device
- **UI Tests:** Espresso (`espresso-core:3.0.2`) — instrumented UI interaction tests

## Test Organization

```
app/src/
├── test/                          # Local JUnit unit tests (JVM)
│   └── java/com/gl/kev/app/
│       └── ExampleUnitTest.kt     # Placeholder; expand here
└── androidTest/                   # Instrumented tests (device/emulator)
    └── java/com/gl/kev/app/
        └── ExampleInstrumentedTest.kt  # Verifies app package name
```

Currently only placeholder tests exist. New tests should follow the structure below.

## Test Naming Conventions

### File Naming

- **Unit test files:** `<ClassUnderTest>Test.kt` (e.g., `AppDataManagerTest.kt`, `MainViewModelTest.kt`)
- **Instrumented test files:** `<Feature>InstrumentedTest.kt` (e.g., `MainActivityInstrumentedTest.kt`)

### Test Method Naming

Use descriptive names that follow the pattern `<method>_<scenario>_<expectedOutcome>`:

```kotlin
@Test
fun getPhotos_onSuccess_callsResponseConsumer()

@Test
fun getPhotos_onNetworkError_callsFailureConsumer()

@Test
fun doGetTodos_returnsObservableOfTodoResponse()
```

## Test Structure (AAA Pattern)

```kotlin
@Test
fun getPhotos_onSuccess_callsResponseConsumer() {
    // Arrange
    val mockApiHelper = mock(ApiHelper::class.java)
    val mockDbHelper = mock(DbHelper::class.java)
    val fakePhotos = PhotoResponse().apply { add(Photo(1, 1, "title", "url", "thumb")) }
    `when`(mockApiHelper.doGetPhotos()).thenReturn(Observable.just(fakePhotos))
    val dataManager = AppDataManager(mockDbHelper, mockApiHelper)

    // Act
    val captured = mutableListOf<PhotoResponse>()
    dataManager.getPhotos(Consumer { captured.add(it) }, Consumer { })

    // Assert
    assertEquals(1, captured.size)
    assertEquals(1, captured[0].size)
}
```

## Mocking Strategy

### Unit Tests

The `SchedulerProvider` interface in `android_framework` is designed to be substituted in tests, enabling synchronous RxJava execution:

```kotlin
class TestSchedulerProvider : SchedulerProvider {
    override fun ui() = Schedulers.trampoline()
    override fun computation() = Schedulers.trampoline()
    override fun io() = Schedulers.trampoline()
}
```

Pass `TestSchedulerProvider()` when constructing `BaseDataManager` subclasses directly or inject via Dagger test modules.

Mock dependencies using Mockito or manual fakes:

```kotlin
val mockApiHelper = mock(ApiHelper::class.java)
val mockDbHelper = mock(DbHelper::class.java)
val dataManager = AppDataManager(mockDbHelper, mockApiHelper)
```

### Dagger Test Components

For ViewModel tests requiring Dagger injection, create a test `ApplicationComponent` that replaces real modules with test doubles:

```kotlin
@Component(modules = [TestRestApiModule::class, TestRoomDataBaseModule::class, ...])
interface TestApplicationComponent : ApplicationComponent
```

### Instrumented Tests

- Use `InstrumentationRegistry.getTargetContext()` for `Context`
- Use Espresso `ActivityScenario` or `ActivityTestRule` for Activity lifecycle
- Room can be tested with an in-memory database: `Room.inMemoryDatabaseBuilder(...)`

## Coverage Targets

No coverage tooling is currently configured. Recommended targets for new code:

| Layer | Target |
|-------|--------|
| `DataManager` / `AppDataManager` | ≥ 80% |
| ViewModels | ≥ 70% |
| DAOs (Room) | ≥ 70% (instrumented) |
| Base classes (`android_framework`) | ≥ 60% |

### Enabling Coverage

Add to `app/build.gradle`:

```groovy
android {
    buildTypes {
        debug {
            testCoverageEnabled true
        }
    }
}
```

Run with: `./gradlew createDebugCoverageReport`

## Running Tests

```bash
# Local unit tests
./gradlew :app:test

# Instrumented tests (requires connected device/emulator)
./gradlew :app:connectedAndroidTest

# All tests
./gradlew test connectedAndroidTest
```

## Key Testability Patterns Already in Place

- **`SchedulerProvider` interface** — swap `AppSchedulerProvider` with `TestSchedulerProvider` (trampoline scheduler) to make RxJava chains synchronous in unit tests
- **Constructor injection via Dagger** — `AppDataManager`, `AppApiHelper`, `AppDbHelper` all use `@Inject` constructors, making them instantiable in tests without Dagger
- **Interface boundaries** — `ApiHelper`, `DbHelper`, `DataManager` are interfaces; any implementation can be substituted in tests

## Best Practices

- Tests should be fast — unit tests must not perform I/O or start the Android framework
- Substitute `AppSchedulerProvider` with `TestSchedulerProvider` (trampoline) in all RxJava unit tests to avoid async timing issues
- For Room DAOs, prefer instrumented tests with an in-memory `AppDataBase` over mocking Room internals
- Keep instrumented tests focused on integration points (database, Activity lifecycle) — not business logic
- `CompositeDisposable` is managed by `BaseDataManager.dispose()` — call this in test teardown to prevent leaks
