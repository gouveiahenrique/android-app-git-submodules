# Code Conventions

**Last updated:** 2026-05-18

## Naming Conventions

### Classes and Interfaces

- `PascalCase` for all classes and interfaces (Kotlin standard)
- Interfaces named by contract role, **no `I` prefix**: `ApiHelper`, `DataManager`, `DbHelper`, `SchedulerProvider`
- Implementations prefixed with scope: `App` for app-specific (`AppApiHelper`, `AppDataManager`), `Base` for reusable generics (`BaseActivity`, `BaseDataManager`)
- Dagger components: suffix `Component` (e.g., `ApplicationComponent`)
- Dagger modules: suffix `Module` (e.g., `RestApiModule`, `RoomDataBaseModule`)
- Room entities: plain model names matching table concept (e.g., `Photo`, `Todo`)
- Room DAOs: suffix `Dao` (e.g., `PhotoDao`, `TodoDao`)
- Room database: suffix `DataBase` (e.g., `AppDataBase`)
- ViewModels: suffix `ViewModel` (e.g., `MainViewModel`, `StartupViewModel`)
- Activities: suffix `Activity` (e.g., `MainActivity`, `StartupActivity`)
- Fragments: suffix `Fragment` (e.g., `WelcomeFragment`, `NextFragment`)
- Coordinators: suffix `Coordinator` (e.g., `RootFlowCoordinator`, `StarupCoordinator`)

### Properties and Functions

- `camelCase` for properties and function names (Kotlin standard)
- `m` prefix for injected or lateinit member fields in the Android layer: `mDataManager`, `mApplicationComponent`, `mBinding`, `mViewModel`
- `get` prefix for data retrieval functions: `getPhotos()`, `getTodos()`, `getDbHelper()`
- `do` prefix for actual network call implementations in helpers: `doGetPhotos()`, `doGetTodos()`

### Constants

- `UPPER_SNAKE_CASE` inside `companion object` (e.g., `DB_NAME`, `API_URL_TODOS`, `DEFAULT_BINDING_VARIABLE`)
- Group related constants in a dedicated `companion object` within a class (e.g., `ApiEndPoint.Companion`, `AppConstants.Companion`)

### Files

- One primary public class per file; file name matches the class name exactly (`MainViewModel.kt`)
- Multiple small related declarations may share a file (e.g., `Responses.kt` contains both `PhotoResponse` and `TodoResponse`)

## Code Formatting

- **Kotlin Code Style:** `official` (configured via `kotlin.code.style=official` in `gradle.properties`)
- **Indentation:** 4 spaces (Kotlin IntelliJ default)
- **Line Length:** ~120 characters (IDE default; no explicit override in this project)
- Use trailing lambda syntax for single-argument lambdas where idiomatic

## Kotlin Language Patterns

### Data Classes for Models

Domain/persistence models are `data class` with `@PrimaryKey` on the `id` field:

```kotlin
@Entity(tableName = "photo")
data class Photo(
    @PrimaryKey val id: Int,
    val albumId: Int,
    var title: String,
    var url: String,
    var thumbnailUrl: String
)
```

`val` for immutable identity fields (`id`, `albumId`), `var` for mutable content fields.

### Companion Objects for Constants

```kotlin
class ApiEndPoint {
    companion object {
        const val API_URL_TODOS  = "${BuildConfig.BASE_ENDPOINT_URL}/todos"
        const val API_URL_PHOTOS = "${BuildConfig.BASE_ENDPOINT_URL}/photos"
    }
}
```

### Generic Base Classes via Reified Type Parameters

`BaseActivity` and `BaseFragment` use reflection on `ParameterizedType` to automatically instantiate the correct `ViewModel` generic type — do not override `initViewModel()`:

```kotlin
abstract class BaseActivity<T : ViewDataBinding, M : AndroidViewModel> : AppCompatActivity() {
    lateinit var mBinding: T
    lateinit var mViewModel: M
    // ViewModel is resolved via reflection on generic type argument M
}
```

Concrete subclasses only implement `initViews()`, `getLayout()`, and `getBindingVariable()`:

```kotlin
class MainActivity : BaseActivity<ActivityMainBinding, MainViewModel>() {
    override fun initViews(savedInstanceState: Bundle?) { ... }
    override fun getLayout() = R.layout.activity_main
    override fun getBindingVariable() = DEFAULT_BINDING_VARIABLE
}
```

## Dependency Injection (Dagger 2)

- Inject all non-trivial dependencies via Dagger; avoid manual instantiation in production code
- Use `@Inject constructor` on implementation classes:
  ```kotlin
  class AppApiHelper @Inject constructor() : ApiHelper { ... }
  class AppDataManager @Inject constructor(
      private val mDbHelper: DbHelper,
      private val mApiHelper: ApiHelper
  ) : DataManager, BaseDataManager() { ... }
  ```
- Use `@Provides` + `@Singleton` in `@Module` classes to bind interfaces to implementations
- Use `@Qualifier` annotations (e.g., `@DatabaseInfo`) to disambiguate multiple bindings of the same type
- ViewModels use field injection (Dagger limitation with `ViewModelProvider`):
  ```kotlin
  class MainViewModel(application: Application) : AndroidViewModel(application) {
      @Inject lateinit var mDataManager: DataManager
      init { getApplication<App>().mApplicationComponent.inject(this) }
  }
  ```

## Reactive Programming (RxJava 2)

All async operations must go through RxJava observables with explicit scheduler management:

```kotlin
// Always subscribeOn io() for background work, observeOn ui() for result delivery
mApiHelper.doGetTodos()
    .subscribeOn(mSchedulerProvider.io())
    .observeOn(mSchedulerProvider.ui())
    .subscribe(response, failure)
```

- Add all subscriptions to `getCompositeDisposable()` via `.add(...)` — never store bare `Disposable` references
- Use `mSchedulerProvider` (injected `SchedulerProvider`) — never reference `Schedulers` or `AndroidSchedulers` directly in `DataManager` or above; this enables test substitution

```kotlin
getCompositeDisposable().add(
    mApiHelper.doGetPhotos()
        .subscribeOn(mSchedulerProvider.io())
        .observeOn(mSchedulerProvider.ui())
        .subscribe(response, failure)
)
```

- Call `dispose()` on `BaseDataManager` when the data manager is no longer needed to prevent leaks

## Data Binding

- Enable Data Binding in each module's `build.gradle` (`dataBinding { enabled = true }`)
- Generated binding classes follow the pattern `Activity<ScreenName>Binding` / `Fragment<ScreenName>Binding`
- ViewModel is bound to layout via `setVariable(bindingVariable, mViewModel)` in base classes
- For screens that don't need two-way binding, return `DEFAULT_BINDING_VARIABLE` (0) from `getBindingVariable()`
- `BindingAdapter` functions for custom attributes live in `android_framework/utils/BindingAdapters.kt`

## Error Handling

- API and database errors propagate as `Consumer<Throwable>` passed to the call site
- Callers are responsible for handling or logging errors:
  ```kotlin
  mDataManager.getPhotos(
      Consumer { /* handle success */ },
      Consumer { Log.e("Tag", it.message, it) }
  )
  ```
- No custom exception hierarchy currently exists — raw `Throwable` is passed through

## Logging

- Use `android.util.Log` with consistent tags matching the feature/class (e.g., `"App"`, `"Demo"`)
- Use `Log.e()` for errors and unexpected states, `Log.i()` for informational milestones
- Wrap verbose logging (e.g., HTTP body logging) behind `BuildConfig.DEBUG` checks:
  ```kotlin
  if (BuildConfig.DEBUG) {
      // AndroidNetworking.enableLogging(HttpLoggingInterceptor.Level.BODY)
  }
  ```

## Comments and Documentation

- Use KDoc (`/** */`) for public API in the `android_framework` module (it is a library consumed externally)
- In-app code uses inline comments only when the behavior is non-obvious
- Include author and date in KDoc for framework base classes (current convention):
  ```kotlin
  /**
   * @author <a href="mailto:...">Name</a>
   * @since MM/DD/YYYY
   */
  ```
- Do not add comments restating what the code already expresses

## Anti-Patterns to Avoid

- **Hardcoded URLs** — always use `ApiEndPoint` constants
- **Direct scheduler references** — never use `Schedulers.io()` or `AndroidSchedulers.mainThread()` outside `AppSchedulerProvider`; always depend on `SchedulerProvider` for testability
- **Skipping `CompositeDisposable`** — never `.subscribe()` without adding to `getCompositeDisposable()`
- **Accessing `mViewModel` before `initViewModel()`** — the base class guarantees initialization order; do not call data methods before `initViews()`
- **Modifying submodule sources directly** — `android_framework` and `android_model` are Git submodules owned externally; changes require a PR to those repos and a submodule pointer update in this repo
- **`fallbackToDestructiveMigration()`** — currently enabled; do not add Room migrations without removing this flag first to avoid silent data loss in production
