# Technology Stack

**Last updated:** 2026-05-18

## Summary

Android application written in Kotlin using a multi-module Gradle project, with the `android_framework` and `android_model` modules managed as Git submodules. The app consumes a REST API, persists data locally with Room, and uses RxJava2 for reactive data streams.

## Languages

- **Primary Language:** Kotlin 1.3.21 (JVM target, JDK 7 stdlib)
- **Secondary Languages:** None (no Java source files)

## Frameworks

### Android

- **UI Framework:** Android SDK (AppCompat, Activities, Fragments)
- **Architecture Pattern:** MVVM — `AndroidViewModel` + `LiveData` + Data Binding
- **Data Binding:** AndroidX Data Binding (`dataBinding { enabled = true }`)
- **Lifecycle:** AndroidX Lifecycle Extensions 2.0.0

### Dependency Injection

- **Framework:** Dagger 2 (2.21) with `kotlin-kapt` for annotation processing
- **Scope:** `@Singleton` for application-level components

### Networking

- **HTTP Client:** RxAndroidNetworking / Rx2AndroidNetworking 1.0.2 (wraps OkHttp)
- **API Base URL:** Configured via `BuildConfig.BASE_ENDPOINT_URL` (`https://jsonplaceholder.typicode.com`)

### Local Storage

- **ORM/Database:** AndroidX Room 2.1.0-alpha05
- **Serialization:** Gson 2.8.5 (used in Room `TypeConverter`)

### Reactive Programming

- **Library:** RxJava2 2.1.12 + RxAndroid 2.0.2
- **Scheduler abstraction:** `SchedulerProvider` interface with `AppSchedulerProvider` implementation

## Major Dependencies

### Runtime Dependencies

| Library | Version | Purpose |
|---------|---------|---------|
| kotlin-stdlib-jdk7 | 1.3.21 | Kotlin standard library |
| androidx.appcompat | 1.1.0-alpha02 | AppCompat Activity/Theme support |
| androidx.lifecycle:lifecycle-extensions | 2.0.0 | ViewModel, LiveData, LifecycleObserver |
| androidx.room:room-runtime | 2.1.0-alpha05 | SQLite ORM |
| io.reactivex.rxjava2:rxjava | 2.1.12 | Reactive streams |
| io.reactivex.rxjava2:rxandroid | 2.0.2 | Android-specific RxJava schedulers |
| com.google.code.gson:gson | 2.8.5 | JSON serialization/deserialization |
| com.google.dagger:dagger | 2.21 | Dependency injection |
| com.amitshekhar.android:rx2-android-networking | 1.0.2 | RxJava2-powered HTTP networking |

### Development / Annotation Processor Dependencies

| Tool | Version | Purpose |
|------|---------|---------|
| androidx.room:room-compiler | 2.1.0-alpha05 | Room DAO/Database code generation |
| androidx.lifecycle:lifecycle-compiler | 2.0.0 | Lifecycle annotation processing |
| com.google.dagger:dagger-compiler | 2.21 | Dagger DI code generation |

### Test Dependencies

| Library | Version | Purpose |
|---------|---------|---------|
| junit:junit | 4.12 | Unit test runner |
| com.android.support.test:runner | 1.0.2 | Android instrumented test runner |
| com.android.support.test.espresso:espresso-core | 3.0.2 | UI testing (Espresso) |

## Build Tools

- **Build System:** Gradle with Android Gradle Plugin 3.4.0
- **Kotlin Plugin:** kotlin-android, kotlin-android-extensions, kotlin-kapt
- **Package Manager:** Gradle (wrapper `gradlew` included)
- **Task Runner:** Gradle tasks (`./gradlew assembleDebug`, `./gradlew test`)

## Development Tools

- **IDE:** Android Studio (recommended; Gradle settings via `org.gradle.jvmargs=-Xmx1536m`)
- **Code Style:** Kotlin official code style (`kotlin.code.style=official`)
- **Linter/Formatter:** No external linter configured; relies on IDE inspections and Kotlin official style
- **Pre-commit Hooks:** None configured

## Runtime Environment

- **Minimum SDK:** API 21 (Android 5.0 Lollipop)
- **Target SDK:** API 28 (Android 9.0 Pie)
- **Compile SDK:** API 28
- **AndroidX:** Enabled (`android.useAndroidX=true`, Jetifier enabled)

## External Services

- **Backend API:** JSONPlaceholder (`https://jsonplaceholder.typicode.com`) — mock REST API used for `todos` and `photos` endpoints
- **Local Database:** SQLite via Room (`app_sample.db`)

## Module Structure

- **`:app`** — Main application module (application plugin)
- **`:android_framework`** — Reusable UI/data framework (library module, Git submodule)
- **`:android_model`** — Shared data models (library module, Git submodule)

## Constraints

- Kotlin 1.3.21 required (pinned in `ext.kotlin_version`)
- Android Gradle Plugin 3.4.0 — older API; does not support configuration caching
- Room version is alpha (`2.1.0-alpha05`); production use requires stable release upgrade
- Git submodules must be initialized before building: `git submodule update --init --recursive`
- `android_framework` and `android_model` are external Git submodules (separate repos); changes must be committed and pushed there independently
