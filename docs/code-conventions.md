# Code Conventions

**Last updated:** 2026-05-18

## Naming Conventions

### Classes and Interfaces

- **PascalCase** for all classes, interfaces, and objects.
- Suffix classes by role:
  - `*Activity` — Android Activity screens
  - `*Fragment` — Android Fragment UI components
  - `*ViewModel` — MVVM ViewModels
  - `*Manager` / `*DataManager` — data layer orchestrators
  - `*Helper` — interface for a specific capability (`ApiHelper`, `DbHelper`)
  - `*Module` — Dagger modules
  - `*Component` — Dagger components
  - `*Coordinator` — navigation coordinators
  - `*Dao` — Room Data Access Objects
  - `*Response` — API response wrappers
  - `Base*` — abstract base classes in the framework module (`BaseActivity`, `BaseDataManager`)

### Functions and Properties

- **camelCase** for functions and properties (`getPhotos()`, `mDataManager`, `mApplicationComponent`).
- Private/injected fields use `m` prefix: `mDataManager`, `mBinding`, `mViewModel` (inherited from Java Android convention).

### Constants

- **UPPER_SNAKE_CASE** inside `companion object` or top-level `const val`:
  ```kotlin
  const val DB_NAME = "app_sample.db"
  const val API_URL_TODOS = "..."
  ```

### Files

- **PascalCase.kt** matching the primary class name (`AppDataManager.kt`, `BaseActivity.kt`).
- One primary class per file; small companion classes (e.g., `PhotoResponse`, `TodoResponse`) may share a file (`Responses.kt`) when they are tightly related.

### Packages

- **Lowercase dot-separated** following reverse-domain convention:
  - App module: `com.gl.kev.app.<layer>` (e.g., `com.gl.kev.app.data.api`)
  - Framework: `com.gl.kev.framework.<layer>`
  - Model: `com.gl.kev.model`

## Code Formatting

### Indentation

- **4 spaces** (Kotlin `official` style, set in `gradle.properties`).
- Do not use tabs.

### Line Length

- **No explicit limit configured.** Follow Kotlin official style guide (≤ 100 characters recommended).

### Braces

- Opening brace on the same line (`K&R style`).
- Empty class bodies (`{ }`) are allowed for marker classes.

## Kotlin Idioms

### Data Classes for Models

All domain entities are `data class`:
```kotlin
data class Photo(
    @PrimaryKey val id: Int,
    val albumId: Int,
    var title: String,
    var url: String,
    var thumbnailUrl: String
)
```
Use `val` for immutable fields (`id`, `albumId`), `var` for fields that may be updated locally.

### Companion Objects for Constants

```kotlin
class ApiEndPoint {
    companion object {
        const val API_URL_TODOS  = "${BuildConfig.BASE_ENDPOINT_URL}/todos"
        const val API_URL_PHOTOS = "${BuildConfig.BASE_ENDPOINT_URL}/photos"
    }
}
```

### Generics in Base Classes

The framework uses generic type parameters to avoid boilerplate:
```kotlin
abstract class BaseActivity<T : ViewDataBinding, M : AndroidViewModel> : AppCompatActivity()
```
Concrete activities supply their binding and ViewModel types at the subclass declaration.

## Dependency Injection (Dagger 2)

- **Constructor injection** is preferred for data/service classes:
  ```kotlin
  class AppApiHelper @Inject constructor() : ApiHelper
  class AppDataManager @Inject constructor(
      private val mDbHelper: DbHelper,
      private val mApiHelper: ApiHelper
  ) : DataManager, BaseDataManager()
  ```
- **Field injection** is used in `ViewModel`s (because ViewModels are instantiated by the framework):
  ```kotlin
  @Inject lateinit var mDataManager: DataManager
  ```
- **Modules** use `@Provides` + `@Singleton` for singleton-scoped objects.
- **Qualifiers** (e.g., `@DatabaseInfo`) differentiate multiple bindings of the same type.

## Reactive Programming (RxJava 2)

- Use `Observable<T>` as the return type from API helpers.
- Always subscribe with both `onNext` and `onError` consumers — never use a single-argument subscribe.
- Add every `Disposable` to the `CompositeDisposable` via `getCompositeDisposable().add(...)` — never hold raw disposables.
- Schedule network operations on `mSchedulerProvider.io()`, observe results on `mSchedulerProvider.ui()`:
  ```kotlin
  mApiHelper.doGetPhotos()
      .subscribeOn(mSchedulerProvider.io())
      .observeOn(mSchedulerProvider.ui())
      .subscribe(response, failure)
  ```
- Call `dispose()` from `BaseDataManager` in `ViewModel.onCleared()` to clean up subscriptions.

## Android Architecture Patterns

### MVVM

- `Activity`/`Fragment` holds only UI interaction code; no business logic.
- `ViewModel` holds state and calls the data layer; no direct Android UI calls.
- `LiveData` is used for DB-backed queries (Room DAOs return `LiveData<T>`).

### Data Binding

- Data Binding is enabled in all modules.
- Activities call `DataBindingUtil.setContentView(this, getLayout())` via `BaseActivity.initBinding()`.
- Bind the ViewModel to the layout via `setVariable(getBindingVariable(), mViewModel)`.

### Coordinator Pattern

- `RootFlowCoordinator` is the single navigation owner; child coordinators (`StartupCoordinator`, `NextCoordinator`) receive navigation lambdas rather than Context references.
- This isolates navigation logic from UI components.

## Error Handling

- API and DB errors are passed as `Consumer<Throwable>` lambdas to data manager methods — never silently swallowed.
- Log errors with `Log.e(TAG, message, throwable)` at the ViewModel layer.
- Do not catch `Exception` broadly; let RxJava propagate errors through the error consumer chain.

## Logging

- Use `Log.e` / `Log.i` with a meaningful tag (typically the class name or `"App"`).
- Debug-only logging should be gated on `BuildConfig.DEBUG`:
  ```kotlin
  if (BuildConfig.DEBUG) {
      // enable verbose network logging
  }
  ```
- Do not leave `Log.d` calls in production paths.

## Comments and Documentation

- **KDoc** is used for public API in the framework module:
  ```kotlin
  /**
   * @author Kevin Villalobos
   * @since 03/12/2019
   */
  ```
- Inline comments explain non-obvious intent, especially around RxJava threading:
  ```kotlin
  // Interrupt — sleep interrupted, treat as completed
  ```
- Do not comment out code — use version control instead.
- `@Suppress` annotations include a reason string when suppressing non-obvious warnings:
  ```kotlin
  @Suppress("UNCHECKED_CAST", "unused")
  ```

## Anti-Patterns to Avoid

- ❌ **Context leaks** — Do not store `Activity` or `Fragment` references in `ViewModel` or singleton objects; use `AndroidViewModel` when `Application` context is needed.
- ❌ **Raw disposables** — Never discard a `Disposable` without adding it to `CompositeDisposable`.
- ❌ **Single-arg subscribe** — `observable.subscribe(consumer)` ignores errors; always provide an error consumer.
- ❌ **Magic strings** — Use `AppConstants` or `companion object` constants instead of inline string literals.
- ❌ **Business logic in Activities/Fragments** — Delegate to ViewModel; keep UI classes thin.
- ❌ **Direct DB calls on main thread** — Always schedule Room/database operations on `mSchedulerProvider.io()`.
- ❌ **Hardcoded URLs** — All API base URLs must go through `BuildConfig.BASE_ENDPOINT_URL` (set in `build.gradle`).
