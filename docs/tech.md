# Technology Stack

**Last updated:** 2026-05-18

## Summary

An Android application written in Kotlin using MVVM architecture, Dagger 2 for DI, RxJava 2 for reactive data streams, Room for local persistence, and the Fast Android Networking library for REST calls. The codebase is split into three Gradle modules managed as Git submodules.

## Languages

- **Primary Language:** Kotlin 1.3.21
- **Secondary Languages:** XML (layouts, manifests, resources)

## Frameworks

### Android SDK

- **Min SDK:** 21 (Android 5.0 Lollipop)
- **Target SDK / Compile SDK:** 28 (Android 9.0 Pie)
- **Architecture Pattern:** MVVM (Model-View-ViewModel) with Coordinator pattern for navigation

### UI

- **AppCompat:** androidx.appcompat 1.1.0-alpha02
- **Data Binding:** Android Data Binding (enabled in all modules)

### Lifecycle / ViewModel

- **Android Lifecycle:** androidx.lifecycle 2.0.0 (LiveData, ViewModel, lifecycle-extensions)

### Dependency Injection

- **Dagger:** 2.21 (dagger-compiler via kapt)

### Reactive Programming

- **RxJava 2:** 2.1.12
- **RxAndroid:** 2.0.2

### Networking

- **Fast Android Networking / Rx2AndroidNetworking:** 1.0.2 (wraps OkHttp, provides RxJava 2 adapters)
- **Base API URL:** `https://jsonplaceholder.typicode.com` (configured via `BuildConfig.BASE_ENDPOINT_URL`)

### Local Storage

- **Room:** 2.1.0-alpha05 (room-runtime + room-compiler via kapt)
- **Database:** SQLite via Room abstraction (entities: `Photo`, `Todo`)

### Serialization

- **Gson:** 2.8.5

## Major Dependencies

### Runtime Dependencies

| Library | Version | Purpose |
|---------|---------|---------|
| kotlin-stdlib-jdk7 | 1.3.21 | Kotlin standard library |
| androidx.appcompat | 1.1.0-alpha02 | AndroidX AppCompat |
| androidx.lifecycle:lifecycle-extensions | 2.0.0 | LiveData + ViewModel |
| androidx.room:room-runtime | 2.1.0-alpha05 | Local SQLite database ORM |
| io.reactivex.rxjava2:rxjava | 2.1.12 | Reactive streams |
| io.reactivex.rxjava2:rxandroid | 2.0.2 | Android thread schedulers for RxJava |
| com.google.dagger:dagger | 2.21 | Dependency injection |
| com.amitshekhar.android:rx2-android-networking | 1.0.2 | HTTP networking with RxJava 2 |
| com.google.code.gson:gson | 2.8.5 | JSON serialization |

### Development / Build Dependencies

| Tool | Version | Purpose |
|------|---------|---------|
| com.android.tools.build:gradle | 3.4.0 | Android Gradle Plugin |
| kotlin-gradle-plugin | 1.3.21 | Kotlin compilation |
| kotlin-kapt | 1.3.21 | Kotlin annotation processing (Dagger, Room) |
| kotlin-android-extensions | 1.3.21 | Synthetic view binding |
| junit | 4.12 | Unit testing |
| espresso-core | 3.0.2 | Instrumented UI testing |
| AndroidJUnitRunner | — | Test runner (instrumented tests) |

## Build Tools

- **Build System:** Gradle (Groovy DSL)
- **Wrapper:** `gradlew` / `gradlew.bat` included in repo
- **Task Runner:** Gradle tasks (`./gradlew assembleDebug`, `./gradlew test`, `./gradlew connectedAndroidTest`)
- **Annotation Processing:** kapt (Kotlin Annotation Processing Tool) for Dagger and Room

## Development Tools

- **IDE:** Android Studio (recommended)
- **Kotlin Style:** `official` (set in `gradle.properties`)
- **Linter:** Android Lint (built-in)
- **Formatter:** Kotlin built-in formatter (Android Studio / `ktlint` not explicitly configured)
- **AndroidX Migration:** Jetifier enabled (`android.enableJetifier=true`)

## Runtime Environment

- **Platform:** Android 5.0+ (API 21+)
- **Permissions:** `android.permission.INTERNET`
- **No Docker / server-side deployment** — pure Android client app

## External Services

- **REST API:** `https://jsonplaceholder.typicode.com` (public mock REST API)
  - `GET /todos` — returns list of Todo objects
  - `GET /photos` — returns list of Photo objects

## Constraints

- Kotlin 1.3.21 required (project was created before coroutines were mainstream; uses RxJava instead)
- Android Gradle Plugin 3.4.0 — must use compatible Gradle wrapper version
- Room 2.1.0-alpha05 is pre-stable; schema export is disabled (`exportSchema = false`)
- `android_framework` and `android_model` are Git submodules — must run `git submodule update --init` after cloning
- AndroidX Jetifier enabled to migrate legacy Support Library references in transitive dependencies
