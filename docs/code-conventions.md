# Code Conventions

**Last updated:** 2026-05-18

## Language and Style Baseline

The project uses Kotlin with official Kotlin code style enforced via `gradle.properties`:

```properties
kotlin.code.style=official
```

All Kotlin code should conform to the [Kotlin Coding Conventions](https://kotlinlang.org/docs/coding-conventions.html).

## Naming Conventions

### Classes and Interfaces

- `PascalCase` for all types: `AppDataManager`, `BaseActivity`, `SchedulerProvider`
- Interfaces have no `I` prefix (e.g., `ApiHelper`, not `IApiHelper`)
- Abstract base classes prefixed with `Base`: `BaseActivity`, `BaseFragment`, `BaseDataManager`
- Dagger modules suffixed with `Module`: `RestApiModule`, `RoomDataBaseModule`
- Dagger components suffixed with `Component`: `ApplicationComponent`
- Room databases suffixed with `DataBase`: `AppDataBase`
- Room DAOs suffixed with `Dao`: `PhotoDao`, `TodoDao`
- ViewModels suffixed with `ViewModel`: `MainViewModel`, `StartupViewModel`

### Functions and Variables

- `camelCase` for all functions and variables: `getPhotos()`, `mDataManager`, `mApplicationComponent`
- Private/protected fields in classes prefixed with `m`: `mBinding`, `mViewModel`, `mDbHelper`
- Companion object constants: `UPPER_SNAKE_CASE` (`DB_NAME`, `DEFAULT_BINDING_VARIABLE`, `API_URL_TODOS`)
- Boolean functions/properties: descriptive without `is`/`has` prefix unless genuinely a state query (`isNetworkConnected()` is acceptable)

### Files

- One primary public class or interface per file, file name matches class name: `AppDataManager.kt`, `BaseActivity.kt`
- Response/related types may share a file when closely coupled: `Responses.kt` contains both `PhotoResponse` and `TodoResponse`

### Packages

- All lowercase, dot-separated: `com.gl.kev.app.data.local.db.dao`
- Organized by layer within each module: `data`, `ui`, `di`, `coordinator`, `utils`

## Code Formatting

- **Indentation:** 4 spaces (Kotlin official style)
- **Line length:** ~100–120 characters; follow IDE defaults
- **Trailing commas:** Not used in the current codebase
- **Braces:** Opening brace on same line as declaration (K&R style)

## Class Structure Patterns

### ViewModel Pattern

ViewModels extend `AndroidViewModel`, use field injection via `@Inject` after Dagger component injection in `init`:

```kotlin
class MainViewModel(application: Application) : AndroidViewModel(application) {

    @Inject
    lateinit var mDataManager: DataManager

    init {
        getApplication<App>().mApplicationComponent.inject(this)
    }

    fun getPhotos() {
        mDataManager.getPhotos(
            Consumer { /* handle success */ },
            Consumer { /* handle error */ }
        )
    }

    override fun onCleared() {
        super.onCleared()
        // dispose subscriptions here if managed outside DataManager
    }
}
```

### Activity Pattern

Activities extend `BaseActivity<BindingType, ViewModelType>` with generic type parameters:

```kotlin
class MainActivity : BaseActivity<ActivityMainBinding, MainViewModel>() {

    override fun initViews(savedInstanceState: Bundle?) {
        // setup views, observe LiveData, trigger initial data loads
    }

    override fun getLayout(): Int = R.layout.activity_main

    override fun getBindingVariable(): Int = DEFAULT_BINDING_VARIABLE
}
```

- `getLayout()` returns the layout resource ID.
- `getBindingVariable()` returns `DEFAULT_BINDING_VARIABLE` (0) when Data Binding variables are not set, or the BR variable ID when binding the ViewModel.
- `initViews()` is the single lifecycle callback for all view setup.

### Fragment Pattern

Fragments extend `BaseFragment<BindingType, ViewModelType>`. The ViewModel is shared with the host activity (via `ViewModelProviders.of(activity!!)`):

```kotlin
class WelcomeFragment : BaseFragment<FragmentWelcomeBinding, StartupViewModel>() {
    override fun initViews() { /* setup fragment-specific views */ }
    override fun getLayout(): Int = R.layout.fragment_welcome
    override fun getBindingVariable(): Int = DEFAULT_BINDING_VARIABLE
}
```

### Interface + Implementation Pattern

All major dependencies are defined as interfaces with a single `App`-scoped implementation. The interface lives alongside the implementation:

```kotlin
// DbHelper.kt — interface
interface DbHelper {
    fun getPhotoDao(): PhotoDao
    fun getTodoDao(): TodoDao
}

// AppDbHelper.kt — implementation
class AppDbHelper @Inject constructor(private val mAppDatabase: AppDataBase) : DbHelper {
    override fun getPhotoDao(): PhotoDao = mAppDatabase.photoDao()
    override fun getTodoDao(): TodoDao = mAppDatabase.todoDao()
}
```

### Dagger Module Pattern

Modules use `@Provides` + `@Singleton` methods with `internal` visibility:

```kotlin
@Module
class RoomDataBaseModule(private val mApplication: Application) {

    @Provides
    @Singleton
    internal fun provideDataManager(appDataManager: AppDataManager): DataManager = appDataManager
}
```

## Data Classes (Models)

Room entities use `data class` with constructor-parameter properties and Room annotations:

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

- `@PrimaryKey` fields use `val` (immutable).
- Mutable fields use `var`.
- No business logic in model classes.

## RxJava Conventions

- All reactive chains must be subscribed on `mSchedulerProvider.io()` and observed on `mSchedulerProvider.ui()`.
- All subscriptions must be added to `getCompositeDisposable()`.
- Response and failure paths are always both provided — never subscribe with a single consumer.
- `dispose()` should be called when the scope owning the `BaseDataManager` is destroyed.

```kotlin
getCompositeDisposable().add(
    observable
        .subscribeOn(mSchedulerProvider.io())
        .observeOn(mSchedulerProvider.ui())
        .subscribe(responseConsumer, failureConsumer)
)
```

## Constants

Application constants go in companion objects of dedicated constants files:

```kotlin
// AppConstants.kt
class AppConstants {
    companion object {
        const val DB_NAME = "app_sample.db"
    }
}
```

API endpoint constants go in `ApiEndPoint.kt` companion object (see `api-standards.md`).

## Comments and Documentation

- Comments in this codebase are mostly KDoc (`/** */`) on public framework classes in `:android_framework`.
- New code in `:app` uses minimal inline comments only where intent is non-obvious.
- KDoc is appropriate on public `BaseActivity`/`BaseFragment` methods explaining their lifecycle contract.
- `TODO:` comments (e.g., `//TODO: test this...` in `GenericListConverter`) should be converted to tracked issues, not left indefinitely.
- Do not write comments that restate what the code already says.

## Suppression Annotations

`@Suppress` is used in framework classes for known Kotlin/IDE warnings:

```kotlin
@Suppress("UNCHECKED_CAST", "unused")
abstract class BaseActivity<T : ViewDataBinding, M : AndroidViewModel> : AppCompatActivity()
```

Use `@Suppress` sparingly and only when the warning is a false positive or an intentional trade-off.

## Anti-Patterns to Avoid

- ❌ **Hardcoded URLs** — always use `ApiEndPoint` constants built from `BuildConfig.BASE_ENDPOINT_URL`
- ❌ **Direct ViewModel injection at field level without the `init` block** — Dagger inject must be called explicitly since `AndroidViewModel` constructors are not Dagger-managed
- ❌ **Leaking subscriptions** — always add disposables to `CompositeDisposable`; call `dispose()` in `onCleared()`
- ❌ **Business logic in Activities or Fragments** — belongs in ViewModel or DataManager
- ❌ **Accessing `android_framework` or `android_model` directly from UI layers** — go through `:app` interfaces (`DataManager`, `DbHelper`)
- ❌ **Modifying submodule files from the root project** — changes to `android_framework` or `android_model` must be committed in their respective repos
- ❌ **Using `ViewModelProviders.of(this)` in Fragments** — use `ViewModelProviders.of(activity!!)` to share the ViewModel with the host Activity (as done in `BaseFragment`)
