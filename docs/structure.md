# Project Structure

**Last updated:** 2026-05-18

## Overview

The project uses a **multi-module MVVM** architecture split across three Gradle modules managed as Git submodules. Separation of concerns is enforced at the module boundary: `android_model` owns domain entities, `android_framework` owns base UI and reactive infrastructure, and `app` owns all application-specific logic (DI wiring, ViewModels, coordinators, data layer, UI screens).

## Directory Layout

```
android-app-git-submodules/
├── app/                            # Application module (main entry point)
│   └── src/
│       ├── androidTest/            # Instrumented tests (Espresso)
│       ├── main/
│       │   ├── java/com/gl/kev/app/
│       │   │   ├── coordinator/    # Navigation coordinators (Coordinator pattern)
│       │   │   ├── data/           # Data layer
│       │   │   │   ├── api/        # Network API helpers and endpoints
│       │   │   │   └── local/db/   # Room database, DAOs, DbHelper
│       │   │   ├── di/             # Dagger 2 DI wiring
│       │   │   │   ├── component/  # Dagger components (ApplicationComponent)
│       │   │   │   ├── module/     # Dagger modules (DB, REST, Coordinator)
│       │   │   │   └── qualifier/  # Dagger qualifiers (@DatabaseInfo)
│       │   │   ├── ui/             # UI screens (Activity + ViewModel pairs)
│       │   │   │   ├── main/       # MainActivity + MainViewModel
│       │   │   │   ├── next/       # NextActivity + NextViewModel
│       │   │   │   └── startup/    # StartupActivity + StartupViewModel + Fragments
│       │   │   └── utils/          # App-level constants (AppConstants)
│       │   ├── AndroidManifest.xml
│       │   └── res/                # Layouts, drawables, strings, styles
│       └── test/                   # Local JVM unit tests
│
├── android_framework/              # [Git submodule] Base framework library
│   └── src/main/java/com/gl/kev/framework/
│       ├── data/                   # BaseDataManager (RxJava + CompositeDisposable)
│       │   └── converter/          # Room TypeConverters (GenericListConverter)
│       ├── ui/                     # BaseActivity<T,M> + BaseFragment<T,M>
│       └── utils/                  # ViewUtils, FontCache, BindingAdapters
│           └── rx/                 # SchedulerProvider + AppSchedulerProvider
│
├── android_model/                  # [Git submodule] Domain model library
│   └── src/main/java/com/gl/kev/model/
│       ├── Photo.kt                # Room @Entity: photo table
│       ├── Todo.kt                 # Room @Entity: todo table
│       └── io/
│           └── Responses.kt        # PhotoResponse, TodoResponse (typed ArrayList wrappers)
│
├── gradle/wrapper/                 # Gradle wrapper files
├── guides/                         # Developer guides (Git submodule setup, research)
├── build.gradle                    # Root build file (version constants, classpath)
├── settings.gradle                 # Module includes: :app, :android_framework, :android_model
├── gradle.properties               # JVM args, AndroidX flags, Kotlin code style
├── .gitmodules                     # Submodule URL mappings
└── README.md
```

## Module Organization

### `:app` — Application Module

| Package | Responsibility |
|---------|---------------|
| `coordinator/` | Navigation flow using the Coordinator pattern; `RootFlowCoordinator` owns child coordinators |
| `data/` | `DataManager` interface + `AppDataManager` implementation; delegates to `ApiHelper` and `DbHelper` |
| `data/api/` | `ApiHelper` interface, `AppApiHelper` (Rx2AndroidNetworking calls), `ApiEndPoint` constants |
| `data/local/db/` | `AppDataBase` (Room), DAOs (`PhotoDao`, `TodoDao`), `DbHelper`/`AppDbHelper` |
| `di/` | Dagger 2 `ApplicationComponent` + three modules: `RoomDataBaseModule`, `RestApiModule`, `CoordinatorModule` |
| `ui/<screen>/` | One sub-package per screen; each contains an `Activity` (or `Fragment`) + `ViewModel` pair |
| `utils/` | `AppConstants` (DB name, etc.) |

### `:android_framework` — Base Framework Library (Git Submodule)

| Package | Responsibility |
|---------|---------------|
| `data/` | `BaseDataManager`: manages `CompositeDisposable`, provides `genericCallable` and `timeOut` helpers, holds `SchedulerProvider` |
| `ui/` | `BaseActivity<T: ViewDataBinding, M: AndroidViewModel>` and `BaseFragment` — generic Data Binding + ViewModel initialization |
| `utils/rx/` | `SchedulerProvider` interface + `AppSchedulerProvider` implementation (io/ui/computation schedulers) |
| `utils/` | `BindingAdapters`, `FontCache`, `ViewUtils` |

### `:android_model` — Domain Model Library (Git Submodule)

| Package | Responsibility |
|---------|---------------|
| root | `Photo` and `Todo` — Kotlin `data class` with Room `@Entity` annotations |
| `io/` | `PhotoResponse` and `TodoResponse` — typed `ArrayList` subclasses used for deserialization |

## File Naming Conventions

- **Kotlin files:** `PascalCase.kt` matching the class name (`AppDataManager.kt`, `BaseActivity.kt`)
- **Layout XML:** `activity_<name>.xml`, `fragment_<name>.xml` (snake_case)
- **Resource files:** `snake_case.xml` (strings, colors, styles, drawables)
- **Packages:** `lowercase.dotted` following reverse-domain convention (`com.gl.kev.app.*`)

## Import Patterns

- **All imports are absolute** — no wildcard imports (Kotlin style)
- **Cross-module imports** use the published module package paths (e.g., `com.gl.kev.framework.*`, `com.gl.kev.model.*`)
- `:app` depends on `:android_model` and `:android_framework` via `api` (transitive) in `app/build.gradle`

## Configuration Files

| File | Purpose |
|------|---------|
| `build.gradle` (root) | AGP + Kotlin plugin classpath; shared version `ext` variables |
| `app/build.gradle` | App-specific dependencies, `applicationId`, `BuildConfig` fields |
| `android_framework/build.gradle` | Framework library dependencies |
| `android_model/build.gradle` | Model library dependencies |
| `settings.gradle` | Declares included Gradle modules |
| `gradle.properties` | JVM heap, AndroidX/Jetifier flags, Kotlin code style |
| `app/proguard-rules.pro` | ProGuard rules (minification disabled in current build types) |
| `.gitmodules` | Submodule remote URLs for `android_framework` and `android_model` |

## Architectural Patterns

- **MVVM:** `Activity`/`Fragment` (View) → `ViewModel` → `DataManager` → `ApiHelper` / `DbHelper`
- **Coordinator Pattern:** `RootFlowCoordinator` owns `StartupCoordinator` and `NextCoordinator`; navigation is delegated via lambda references
- **Dependency Injection:** Dagger 2 with constructor injection (`@Inject`) on data layer; field injection into `ViewModel`s via the `ApplicationComponent`
- **Reactive Streams:** All async operations use RxJava 2 `Observable` + `CompositeDisposable` for subscription management; threading handled by `SchedulerProvider`
- **Repository-style Data Layer:** `AppDataManager` unifies `ApiHelper` (network) and `DbHelper` (database) behind the `DataManager` interface
