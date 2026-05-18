# Project Structure

**Last updated:** 2026-05-18

## Overview

Multi-module Android project following a layered architecture (Data / Domain / Presentation). The `:android_framework` and `:android_model` modules are Git submodules, allowing independent versioning and reuse across projects. The `:app` module owns the application entry point, DI wiring, and all screen-specific code.

## Directory Layout

```
android-app-git-submodules/
├── app/                        # Application module (:app)
│   └── src/
│       ├── main/
│       │   ├── java/com/gl/kev/app/
│       │   │   ├── coordinator/    # Flow coordinators (navigation logic)
│       │   │   ├── data/           # Data layer (API, DB, DataManager)
│       │   │   │   ├── api/        # REST API interface + implementation
│       │   │   │   └── local/db/   # Room database, DAOs, DbHelper
│       │   │   ├── di/             # Dagger DI (component + modules + qualifiers)
│       │   │   │   ├── component/
│       │   │   │   ├── module/
│       │   │   │   └── qualifier/
│       │   │   ├── ui/             # UI screens (Activity + ViewModel pairs)
│       │   │   │   ├── main/
│       │   │   │   ├── next/
│       │   │   │   └── startup/    # Launch screen (Activity + Fragments)
│       │   │   └── utils/          # App-level constants
│       │   ├── res/                # Layouts, drawables, values
│       │   └── AndroidManifest.xml
│       ├── test/                   # JVM unit tests
│       └── androidTest/            # Instrumented tests
├── android_framework/          # Git submodule (:android_framework)
│   └── src/main/java/com/gl/kev/framework/
│       ├── data/               # BaseDataManager, RxJava helpers, TypeConverters
│       │   └── converter/
│       ├── ui/                 # BaseActivity<T,M>, BaseFragment<T,M>
│       └── utils/              # BindingAdapters, FontCache, ViewUtils, rx schedulers
├── android_model/              # Git submodule (:android_model)
│   └── src/main/java/com/gl/kev/model/
│       ├── Photo.kt            # Room entity
│       ├── Todo.kt             # Room entity
│       └── io/
│           └── Responses.kt    # PhotoResponse, TodoResponse (ArrayList subclasses)
├── gradle/wrapper/             # Gradle wrapper files
├── guides/                     # Developer documentation (git submodules how-to)
├── build.gradle                # Root build file (versions defined in ext {})
├── settings.gradle             # Module inclusions
├── gradle.properties           # JVM args, AndroidX flags, Kotlin code style
├── gradlew / gradlew.bat       # Gradle wrapper scripts
└── .gitmodules                 # Git submodule configuration
```

## Module Organization

### `:app` — Application Module

Owns application lifecycle, DI wiring, and all feature screens.

- **coordinator/** — Flow coordinator pattern for navigation. `RootFlowCoordinator` aggregates child coordinators (`StarupCoordinator`, `NextCoordinator`). `Navigator` is a `@Singleton` helper for activity transitions.
- **data/api/** — `ApiHelper` interface defines reactive API calls; `AppApiHelper` implements using `Rx2AndroidNetworking`. `ApiEndPoint` holds URL constants sourced from `BuildConfig`.
- **data/local/db/** — `AppDataBase` (`RoomDatabase` subclass), `DbHelper` interface, `AppDbHelper` implementation, and DAO interfaces (`PhotoDao`, `TodoDao`).
- **data/** — `DataManager` interface combines `ApiHelper` and `DbHelper` access; `AppDataManager` implements both, extending `BaseDataManager` for RxJava disposable management.
- **di/component/** — `ApplicationComponent` (Dagger `@Singleton` component) with `inject()` methods for `App` and each `ViewModel`.
- **di/module/** — `RoomDataBaseModule` (provides `AppDataBase`, `DataManager`, `DbHelper`), `RestApiModule` (provides `ApiHelper`), `CoordinatorModule` (provides coordinators).
- **di/qualifier/** — Custom Dagger `@Qualifier` annotations (e.g., `@DatabaseInfo` for the DB name string).
- **ui/** — Each screen has an `Activity` (extends `BaseActivity<Binding, ViewModel>`) and a paired `ViewModel` (extends `AndroidViewModel`). Fragments extend `BaseFragment<Binding, ViewModel>`.
- **utils/** — `AppConstants` (companion object with DB name constant).

### `:android_framework` — Shared Framework (Git Submodule)

Reusable Android framework providing base classes and utilities.

- **data/BaseDataManager** — Manages `CompositeDisposable`, exposes `SchedulerProvider`, provides `genericCallable()` helper and `timeOut()` utility.
- **data/converter/GenericListConverter** — Room `@TypeConverter` using Gson for `List<T>` ↔ JSON.
- **ui/BaseActivity<T, M>** — Generic `AppCompatActivity` subclass; auto-inflates Data Binding layout via `DataBindingUtil`, auto-resolves `AndroidViewModel` subclass via reflection.
- **ui/BaseFragment<T, M>** — Generic `Fragment` subclass; mirrors `BaseActivity` pattern; shares `ViewModel` with host activity via `ViewModelProviders.of(activity!!)`.
- **utils/** — `BindingAdapters` (custom `@BindingAdapter` for font and visibility), `FontCache` (typeface caching), `ViewUtils` (color resolution helper).
- **utils/rx/** — `SchedulerProvider` interface + `AppSchedulerProvider` (wraps `AndroidSchedulers.mainThread()`, `Schedulers.io()`, `Schedulers.computation()`).

### `:android_model` — Shared Models (Git Submodule)

Pure data model library with no Android framework dependencies beyond Room annotations.

- **Photo.kt** — `@Entity(tableName = "photo")` data class with `@PrimaryKey`.
- **Todo.kt** — `@Entity(tableName = "todo")` data class with `@PrimaryKey`.
- **io/Responses.kt** — `PhotoResponse` and `TodoResponse` extend `ArrayList<Photo>` and `ArrayList<Todo>` respectively (used as deserialization targets for the REST API).

## File Naming Conventions

- **Kotlin source files:** `PascalCase.kt` matching the primary class name (e.g., `AppDataManager.kt`, `BaseActivity.kt`)
- **Layout XML:** `activity_<screen>.xml`, `fragment_<name>.xml` (snake_case)
- **Test files:** `Example*Test.kt` — no project-specific naming convention beyond the Gradle default

## Package Structure

All packages follow the pattern `com.gl.kev.<module>.<layer>`:

- `com.gl.kev.app` — Application root
- `com.gl.kev.app.data` — Data layer
- `com.gl.kev.app.di` — Dependency injection
- `com.gl.kev.app.ui` — Presentation layer
- `com.gl.kev.app.coordinator` — Navigation/flow
- `com.gl.kev.framework` — Shared framework
- `com.gl.kev.model` — Shared models

## Architectural Patterns

- **MVVM:** Activities/Fragments own the view; `AndroidViewModel` subclasses own presentation logic and data fetching; Data Binding connects them declaratively.
- **Repository-like DataManager:** `DataManager` interface unifies API and DB access; `AppDataManager` is the single implementation injected into ViewModels.
- **Interface Segregation:** `ApiHelper`, `DbHelper`, `DataManager`, `SchedulerProvider`, and `Navigator` are all defined as interfaces — implementations are swapped via Dagger.
- **Coordinator Pattern:** `RootFlowCoordinator` delegates to child coordinators per-flow; intended to decouple navigation from Activities (partially implemented).
- **Git Submodules:** `:android_framework` and `:android_model` live in separate repositories and are included as git submodules, enabling reuse across multiple apps.

## Configuration Files

- **Root `build.gradle`** — Android Gradle Plugin version, Kotlin version, shared version constants in `ext {}`
- **`settings.gradle`** — Module declarations (`:app`, `:android_framework`, `:android_model`)
- **`gradle.properties`** — JVM heap, AndroidX/Jetifier flags, Kotlin code style
- **`app/build.gradle`** — Application plugin config, `BuildConfig` fields (`BASE_ENDPOINT_URL`), full dependency list
- **`AndroidManifest.xml`** — `INTERNET` permission, Activity declarations, application class
- **`.gitmodules`** — Submodule remote URLs for `android_framework` and `android_model`
