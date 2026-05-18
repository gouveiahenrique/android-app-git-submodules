# Technology Stack

**Last updated:** 2026-05-18

## Summary

Android application written in Kotlin using the MVVM architecture pattern, structured as a multi-module Gradle project with two Git submodules (`android_framework`, `android_model`) providing reusable base classes and data models.

## Languages

- **Primary Language:** Kotlin 1.3.21 (targeting JVM/Android)
- **Build Scripting:** Groovy DSL (Gradle)

## Frameworks

### Android Platform

- **Min SDK:** API 21 (Android 5.0 Lollipop)
- **Target SDK:** API 28 (Android 9.0 Pie)
- **Compile SDK:** API 28

### Architecture & UI

- **UI Pattern:** MVVM with Android Architecture Components
- **Data Binding:** AndroidX Data Binding (enabled in all modules)
- **ViewModel:** AndroidX Lifecycle `lifecycle-extensions:2.0.0`
- **Navigation Pattern:** Coordinator pattern (custom `RootFlowCoordinator`, `StarupCoordinator`, `NextCoordinator`)

### Dependency Injection

- **DI Framework:** Dagger 2 (`dagger:2.21`)
- **Annotation Processing:** `kotlin-kapt` plugin

### Networking

- **HTTP Client:** `rx2-android-networking:1.0.2` (RxJava 2 Android Networking by AmitShekhar, wraps OkHttp)
- **Base URL:** Configured via `BuildConfig.BASE_ENDPOINT_URL` (default: `https://jsonplaceholder.typicode.com`)

### Reactive Programming

- **Reactive Streams:** RxJava 2 (`rxjava:2.1.12`)
- **Android RxJava Bridge:** RxAndroid (`rxandroid:2.0.2`)

### Local Persistence

- **ORM/Database:** AndroidX Room (`room-runtime:2.1.0-alpha05`)
- **Entities:** `Photo`, `Todo` (SQLite via Room DAOs with `LiveData` return types)

### Serialization

- **JSON:** Gson (`gson:2.8.5`) — used for HTTP response deserialization

## Major Dependencies

### Runtime Dependencies

| Library | Version | Purpose |
|---------|---------|---------|
| kotlin-stdlib-jdk7 | 1.3.21 | Kotlin standard library |
| androidx.appcompat | 1.1.0-alpha02 | AppCompat/Material base UI |
| lifecycle-extensions | 2.0.0 | ViewModel, LiveData, AndroidViewModel |
| room-runtime | 2.1.0-alpha05 | Local SQLite database via Room |
| dagger | 2.21 | Compile-time dependency injection |
| rxjava | 2.1.12 | Reactive programming streams |
| rxandroid | 2.0.2 | RxJava Android thread schedulers |
| rx2-android-networking | 1.0.2 | Reactive HTTP networking |
| gson | 2.8.5 | JSON serialization/deserialization |

### Development / Annotation-Processor Dependencies

| Tool | Version | Purpose |
|------|---------|---------|
| room-compiler | 2.1.0-alpha05 | Room DAO annotation processor (kapt) |
| lifecycle-compiler | 2.0.0 | Lifecycle annotation processor (kapt) |
| dagger-compiler | 2.21 | Dagger code generation (kapt) |

### Test Dependencies

| Tool | Version | Purpose |
|------|---------|---------|
| junit | 4.12 | Local unit tests |
| androidx.test runner | 1.0.2 | Android instrumented test runner |
| espresso-core | 3.0.2 | Android UI instrumented tests |

## Build Tools

- **Build System:** Gradle 5.1.1 (via Gradle Wrapper)
- **Android Gradle Plugin:** 3.4.0
- **Kotlin Gradle Plugin:** 1.3.21
- **Kotlin Plugins:** `kotlin-android`, `kotlin-android-extensions`, `kotlin-kapt`

## Module Structure

- **`:app`** — `com.android.application` — main application module
- **`:android_framework`** — `com.android.library` — Git submodule; base classes (BaseActivity, BaseFragment, BaseDataManager, RxSchedulers)
- **`:android_model`** — `com.android.library` — Git submodule; data models (Photo, Todo) and response wrappers

## Runtime Environment

- **Platform:** Android (minSdk 21+)
- **AndroidX:** Enabled (`android.useAndroidX=true`, Jetifier enabled)
- **Kotlin Code Style:** Official (`kotlin.code.style=official`)
- **Database File:** `app_sample.db` (Room SQLite, local on-device)

## External Services

- **REST API:** JSONPlaceholder (`https://jsonplaceholder.typicode.com`) — public mock REST API
  - `GET /todos` — returns list of `Todo` objects
  - `GET /photos` — returns list of `Photo` objects

## Constraints

- Kotlin 1.3.21 required (project was written against this version; newer versions should be backwards-compatible but untested)
- Gradle 5.1.1 is pinned via wrapper — do not upgrade without verifying AGP 3.4.0 compatibility
- Room version is alpha (`2.1.0-alpha05`) — API may not be stable
- `android_framework` and `android_model` are Git submodules at fixed commits; changes require updating the submodule reference
- `fallbackToDestructiveMigration()` is enabled on the Room database — schema migrations will drop and recreate the database
