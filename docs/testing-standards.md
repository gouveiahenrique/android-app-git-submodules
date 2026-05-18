# Testing Standards

**Last updated:** 2026-05-18

## Testing Frameworks

- **Unit Tests (JVM):** JUnit 4.12 — runs on the development machine, no Android device needed
- **Instrumented Tests (Android):** AndroidJUnitRunner (`androidx.test.runner.AndroidJUnitRunner`) + Espresso 3.0.2 — runs on a device or emulator

## Test Organization

```
app/src/
├── test/java/com/gl/kev/app/          # JVM unit tests (fast, no Android runtime)
│   └── ExampleUnitTest.kt
└── androidTest/java/com/gl/kev/app/   # Instrumented tests (device/emulator required)
    └── ExampleInstrumentedTest.kt
```

Both `android_framework` and `android_model` submodules currently contain no test sources. New tests for submodule logic should be added to the submodule's own `src/test/` directory.

## Current Test Coverage

The project contains only auto-generated scaffold tests (no real business logic coverage):

- `ExampleUnitTest` — trivial JUnit assertion (`2 + 2 == 4`)
- `ExampleInstrumentedTest` — verifies the application package name

Test coverage must be expanded as features are added. Priority areas for new tests:

1. `AppDataManager` — data fetching, RxJava threading, disposable management
2. `AppApiHelper` — API call construction and observable return types
3. Room DAOs — insert/query/delete operations
4. `BaseDataManager` — `CompositeDisposable` lifecycle, `genericCallable`, `timeOut`

## Test Naming Conventions

### File Naming

- Unit test files: `<ClassUnderTest>Test.kt` (e.g., `AppDataManagerTest.kt`)
- Instrumented test files: `<ClassUnderTest>InstrumentedTest.kt` or `<Screen>Test.kt`

### Method Naming

Use descriptive names that read as a statement of what is being verified:

```kotlin
// Good
@Test fun getPhotos_callsApiHelperAndDeliversResult()
@Test fun dispose_clearsCompositeDisposable()

// Avoid
@Test fun test1()
@Test fun testGetPhotos()
```

## Test Structure (AAA Pattern)

All tests should follow Arrange → Act → Assert:

```kotlin
@Test
fun getPhotos_subscribesOnIoAndObservesOnUi() {
    // Arrange
    val mockApiHelper = mock(ApiHelper::class.java)
    val mockDbHelper = mock(DbHelper::class.java)
    val testScheduler = TestScheduler()
    val schedulerProvider = TestSchedulerProvider(testScheduler)
    val dataManager = AppDataManager(mockDbHelper, mockApiHelper)

    // Act
    var result: PhotoResponse? = null
    dataManager.getPhotos(Consumer { result = it }, Consumer { })

    // Assert
    assertNotNull(result)
}
```

## Mocking Strategy

### Unit Tests

- Use constructor injection (all dependencies are interfaces) to inject mocks.
- Recommended mocking library: **Mockito** or **MockK** (neither is currently in `build.gradle` — add to `testImplementation` before writing mocked tests).
- For RxJava testing, use `TestScheduler` via a test implementation of `SchedulerProvider`:

```kotlin
class TestSchedulerProvider(private val scheduler: TestScheduler) : SchedulerProvider {
    override fun ui() = scheduler
    override fun io() = scheduler
    override fun computation() = scheduler
}
```

### Instrumented Tests (Espresso)

- Use the real application (`App`) with a test application class that swaps the Dagger component for a test component where needed.
- Espresso should interact with UI elements via resource IDs or content descriptions, not positional accessors.

## Running Tests

```bash
# JVM unit tests
./gradlew test

# Instrumented tests (requires connected device or emulator)
./gradlew connectedAndroidTest

# Run tests for a specific module
./gradlew :app:test
./gradlew :android_framework:test
```

## CI/CD Integration

No CI/CD pipeline is currently configured. When adding one (e.g., GitHub Actions), the recommended setup is:

```yaml
- name: Run unit tests
  run: ./gradlew test

- name: Run instrumented tests (emulator)
  uses: reactivecircus/android-emulator-runner@v2
  with:
    api-level: 28
    script: ./gradlew connectedAndroidTest
```

## Best Practices

- ✅ Unit tests should run without a device — no Android framework classes in pure logic tests
- ✅ Use `SchedulerProvider` interface to inject `TestScheduler` in RxJava tests
- ✅ Use Dagger injection with interfaces — makes swapping real implementations for mocks straightforward
- ✅ Keep tests independent — each test sets up its own state, no shared mutable fields between tests
- ✅ Test the `DataManager` layer separately from `ViewModel` and API layers
- ❌ Do not test Dagger-generated code (component wiring) — test behavior, not framework boilerplate
- ❌ Do not make real network calls in unit or CI tests — mock `ApiHelper`
- ❌ Do not leave `TODO: test this...` comments without a corresponding test task (see `GenericListConverter`)
