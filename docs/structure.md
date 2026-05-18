# Project Structure

**Last updated:** 2026-05-18

## Overview

Multi-module Android Gradle project with MVVM architecture. The codebase is split into three Gradle modules: a main `:app` module and two Android library modules (`:android_framework`, `:android_model`) that are managed as Git submodules. The `:app` module follows a layered package structure: `data` (repositories/API/DB), `di` (Dagger), `ui` (Activities/Fragments/ViewModels), `coordinator` (navigation), and `utils`.

## Directory Layout

```
project-root/
├── app/                            # Main application module (:app)
│   └── src/
│       ├── main/
│       │   └── java/com/gl/kev/app/
│       │       ├── coordinator/    # Navigation coordinators
│       │       ├── data/           # Data layer (API, DB, DataManager)
│       │       │   ├── api/        # REST API helper and endpoints
│       │       │   └── local/db/   # Room database, DAOs
│       │       ├── di/             # Dagger 2 dependency injection
│       │       │   ├── component/  # Dagger @Component interfaces
│       │       │   ├── module/     # Dagger @Module providers
│       │       │   └── qualifier/  # Dagger @Qualifier annotations
│       │       ├── ui/             # UI layer (MVVM screens)
│       │       │   ├── main/       # MainActivity + MainViewModel
│       │       │   ├── next/       # NextActivity + NextViewModel
│       │       │   └── startup/    # StartupActivity + ViewModels + Fragments
│       │       └── utils/          # AppConstants
│       ├── test/                   # Local JUnit unit tests
│       └── androidTest/            # Instrumented Espresso tests
│
├── android_framework/              # Git submodule: base library (:android_framework)
│   └── src/main/java/com/gl/kev/framework/
│       ├── data/                   # BaseDataManager (RxJava disposable management)
│       ├── ui/                     # BaseActivity<T, M>, BaseFragment<T, M>
│       └── utils/                  # ViewUtils, FontCache, BindingAdapters, RxSchedulers
│
├── android_model/                  # Git submodule: model library (:android_model)
│   └── src/main/java/com/gl/kev/model/
│       ├── Photo.kt                # Room @Entity: photo table
│       ├── Todo.kt                 # Room @Entity: todo table
│       └── io/
│           └── Responses.kt        # PhotoResponse, TodoResponse (ArrayList wrappers)
│
├── gradle/wrapper/                 # Gradle wrapper (Gradle 5.1.1)
├── guides/                         # Developer guides (git submodule how-to)
├── docs/                           # Steering documentation (this directory)
├── build.gradle                    # Root build file (ext versions, repositories)
├── settings.gradle                 # Module declarations: app, android_framework, android_model
├── gradle.properties               # JVM args, AndroidX/Jetifier flags, Kotlin code style
└── .gitmodules                     # Git submodule config (android_framework, android_model)
```

## Module Organization

### `:app` — Application Module

#### `coordinator/`
Handles navigation flow between screens using the Coordinator pattern.
- `Navigator` — navigation interface/contract
- `RootFlowCoordinator` — top-level coordinator; owns `StarupCoordinator` and `NextCoordinator`
- `StarupCoordinator` / `NextCoordinator` — screen-specific flow logic

#### `data/`
All data access is centralized through the `DataManager` interface.
- `DataManager` (interface) — public contract for all data operations
- `AppDataManager` (impl) — delegates to `ApiHelper` (network) and `DbHelper` (database); extends `BaseDataManager` for RxJava lifecycle
- `api/ApiHelper` (interface) + `AppApiHelper` (impl) — RxJava Observables for network calls
- `api/ApiEndPoint` — central URL constants derived from `BuildConfig.BASE_ENDPOINT_URL`
- `local/db/AppDataBase` — Room database declaration (`Photo`, `Todo` entities)
- `local/db/DbHelper` (interface) + `AppDbHelper` (impl) — DAO access
- `local/db/dao/PhotoDao`, `TodoDao` — Room DAO interfaces (LiveData queries, insert, update, delete)

#### `di/`
- `component/ApplicationComponent` — singleton Dagger component; injects into `App`, `MainViewModel`, `NextViewModel`, `StartupViewModel`
- `module/RoomDataBaseModule` — provides `AppDataBase`, `DbHelper`, `DataManager`
- `module/RestApiModule` — provides `ApiHelper`
- `module/CoordinatorModule` — provides coordinator dependencies
- `qualifier/DatabaseInfo` — custom `@Qualifier` for database name string

#### `ui/`
Each screen is a self-contained package with an Activity and ViewModel pair.
- `main/` — `MainActivity` + `MainViewModel`
- `next/` — `NextActivity` + `NextViewModel`
- `startup/` — `StartupActivity` + `StartupViewModel`, with `WelcomeFragment` and `NextFragment`

#### `utils/`
- `AppConstants` — application-wide constants (e.g., `DB_NAME = "app_sample.db"`)

### `:android_framework` — Base Library (Git Submodule)

Reusable generic base classes shared across apps.
- `BaseActivity<T : ViewDataBinding, M : AndroidViewModel>` — initializes ViewModel via reflection, sets up Data Binding, provides network check and alert dialog utilities
- `BaseFragment<T : ViewDataBinding, M : AndroidViewModel>` — mirrors BaseActivity for Fragments; shares ViewModel with host Activity
- `BaseDataManager` — manages `CompositeDisposable`, injects `SchedulerProvider`; provides `genericCallable()` helper for background→main-thread RxJava chains
- `AppSchedulerProvider` / `SchedulerProvider` — injectable RxJava scheduler abstraction (io, computation, main thread) enabling test substitution

### `:android_model` — Data Model Library (Git Submodule)

Plain data classes shared across layers.
- `Photo` — Room `@Entity`, data class with `id`, `albumId`, `title`, `url`, `thumbnailUrl`
- `Todo` — Room `@Entity`, data class with `id`, `userId`, `title`, `completed`
- `io/Responses.kt` — `PhotoResponse : ArrayList<Photo>`, `TodoResponse : ArrayList<Todo>` (Gson-deserializable list wrappers)

## File Naming Conventions

- **Kotlin source files:** `PascalCase.kt` matching the primary class name (e.g., `MainViewModel.kt`, `AppDataManager.kt`)
- **Interface files:** Named by contract, not "I" prefix (e.g., `ApiHelper`, `DataManager`, `DbHelper`)
- **Implementation files:** Prefixed with `App` for application-specific impls (e.g., `AppApiHelper`, `AppDataManager`)
- **Test files:** Suffix `Test` for unit tests, `InstrumentedTest` for Espresso tests (e.g., `ExampleUnitTest.kt`)
- **Layout XML files:** `snake_case` matching the screen (e.g., `activity_main.xml`)

## Import Patterns

- Fully-qualified package imports (no wildcard imports)
- Module dependencies declared via `api project(":android_model")` and `api project(":android_framework")` in `:app`'s `build.gradle`; downstream consumers of `:app` inherit these transitively

## Configuration Files

| File | Purpose |
|------|---------|
| `build.gradle` (root) | AGP classpath, Kotlin plugin, shared `ext` version variables |
| `build.gradle` (per-module) | Module-specific plugins and dependencies |
| `settings.gradle` | Declares all included Gradle modules |
| `gradle.properties` | JVM heap, AndroidX/Jetifier flags, Kotlin code style |
| `gradle/wrapper/gradle-wrapper.properties` | Pins Gradle 5.1.1 |
| `.gitmodules` | Registers `android_framework` and `android_model` as Git submodules |
| `proguard-rules.pro` | Module-specific ProGuard rules (minification disabled in current config) |

## Architectural Patterns

- **MVVM:** `Activity`/`Fragment` → `ViewModel` → `DataManager`; UI observes `LiveData` from Room DAOs
- **Repository Pattern:** `DataManager` abstracts API and DB behind a single interface
- **Coordinator Pattern:** Navigation logic extracted from Activities into Coordinator classes
- **Dependency Injection:** Constructor injection via Dagger 2; `@Singleton` scope at application level
- **Reactive Streams:** All async operations (network, background DB writes) use RxJava 2 `Observable` chains with `CompositeDisposable` lifecycle management
