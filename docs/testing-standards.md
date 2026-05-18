# Testing Standards

**Last updated:** 2026-05-18

## Testing Frameworks

| Test Type | Framework | Runner |
|-----------|-----------|--------|
| Local unit tests | JUnit 4.12 | JVM (no device required) |
| Instrumented tests | Espresso 3.0.2 | `AndroidJUnitRunner` on device/emulator |

## Test Organization

```
app/src/
├── test/java/com/gl/kev/app/          # Local JVM unit tests
│   └── ExampleUnitTest.kt
└── androidTest/java/com/gl/kev/app/   # Instrumented (on-device) tests
    └── ExampleInstrumentedTest.kt
```

- **Local unit tests** (`src/test/`) run on the JVM. No Android framework dependencies.
- **Instrumented tests** (`src/androidTest/`) run on an Android device or emulator. Use for testing Android-specific behavior (database, UI, `Context`).

## Test Naming Conventions

### Kotlin (JUnit 4)

```kotlin
// File: ExampleUnitTest.kt
// Method names use snake_case to read as sentences:
fun addition_isCorrect()
fun getPhotos_returnsExpectedCount()
fun insertPhoto_replacesOnConflict()
```

Standard pattern: `methodUnderTest_stateOrInput_expectedBehavior`

## Test Structure (AAA Pattern)

```kotlin
@Test
fun getPhotos_onSuccess_logsCount() {
    // Arrange
    val mockApiHelper = mock(ApiHelper::class.java)
    val mockDbHelper = mock(DbHelper::class.java)
    `when`(mockApiHelper.doGetPhotos()).thenReturn(Observable.just(fakePhotos))

    // Act
    val dataManager = AppDataManager(mockDbHelper, mockApiHelper)
    dataManager.getPhotos(Consumer { result ->
        // Assert
        assertEquals(3, result.size)
    }, Consumer { fail("Should not error") })
}
```

## Running Tests

```bash
# Local unit tests
./gradlew test

# Instrumented tests (requires connected device or emulator)
./gradlew connectedAndroidTest

# Single module
./gradlew :app:test
./gradlew :app:connectedAndroidTest
```

## Mocking Strategy

### Unit Tests

- **Mock external dependencies**: `ApiHelper`, `DbHelper`, any `SchedulerProvider`.
- For RxJava testing, replace `AppSchedulerProvider` with a `TestSchedulerProvider` (Schedulers.trampoline()) to make async observable chains synchronous.
- Use constructor injection (already in place via Dagger) to inject mocks without reflection.

```kotlin
// Inject synchronous schedulers for unit testing
class TestSchedulerProvider : SchedulerProvider {
    override fun ui()          = Schedulers.trampoline()
    override fun io()          = Schedulers.trampoline()
    override fun computation() = Schedulers.trampoline()
}
```

### Instrumented Tests

- Use **real Room database** with an in-memory instance:
  ```kotlin
  Room.inMemoryDatabaseBuilder(context, AppDataBase::class.java).build()
  ```
- Use `Espresso` for UI interaction; avoid mocking Android framework classes.

## Coverage Targets

No explicit coverage threshold is enforced in the current build configuration. Recommended targets for production use:

| Area | Target |
|------|--------|
| `AppDataManager` / data layer | ≥ 80% |
| DAO operations (Room) | ≥ 80% |
| ViewModels | ≥ 70% |
| Coordinators | ≥ 60% |

## CI/CD Integration

No CI/CD pipeline is configured in this repository. To add GitHub Actions:

```yaml
name: Android CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          submodules: recursive        # Required — fetches android_framework and android_model
      - uses: actions/setup-java@v3
        with:
          java-version: '11'
          distribution: 'temurin'
      - name: Run unit tests
        run: ./gradlew test
```

> **Important:** Always use `submodules: recursive` in checkout steps — the project will not compile without the two Git submodules.

## Test Data Management

- No shared fixture files exist; test data is constructed inline in each test.
- For Room integration tests, seed the in-memory database in a `@Before` method and clear it in `@After`.

## Best Practices

- ✅ Prefer local unit tests over instrumented tests where the Android framework is not needed.
- ✅ Use `Schedulers.trampoline()` to make RxJava chains synchronous in unit tests.
- ✅ Test observable chains using `TestObserver` from the `rxjava` test package.
- ✅ Use `OnConflictStrategy.REPLACE` in DAO insert tests to verify idempotency.
- ❌ Do not test private/internal Dagger-generated classes; test through public interfaces.
- ❌ Do not use `Thread.sleep()` in tests — use `TestScheduler` or `trampoline()` instead.
