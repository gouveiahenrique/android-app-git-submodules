# API Standards

**Last updated:** 2026-05-18

## Overview

The app consumes an external REST API (currently JSONPlaceholder at `https://jsonplaceholder.typicode.com`). All network calls are made through `AppApiHelper`, which wraps `Rx2AndroidNetworking`. The base URL is injected at build time via `BuildConfig.BASE_ENDPOINT_URL`, allowing it to be changed per build variant.

The app is a **client only** — it does not expose any API. These standards describe how to write client-side API code consistently.

## Endpoint Definitions

Endpoints are defined as `const val` strings in `ApiEndPoint` companion object, always constructed from `BuildConfig.BASE_ENDPOINT_URL`:

```kotlin
// app/src/main/java/com/gl/kev/app/data/api/ApiEndPoint.kt
class ApiEndPoint {
    companion object {
        const val API_URL_TODOS = "${BuildConfig.BASE_ENDPOINT_URL}/todos"
        const val API_URL_PHOTOS = "${BuildConfig.BASE_ENDPOINT_URL}/photos"
    }
}
```

All new endpoints must be added here. Never hardcode URLs inline in `AppApiHelper` or elsewhere.

## HTTP Methods in Use

| Method | Usage |
|--------|-------|
| GET | Fetching todos and photos lists (only method currently used) |

All current API calls are GET requests. Future POST/PUT/DELETE calls should follow standard REST conventions for the resource being operated on.

## Request Format

All requests go through `Rx2AndroidNetworking` builder:

```kotlin
Rx2AndroidNetworking.get(ApiEndPoint.API_URL_PHOTOS)
    .doNotCacheResponse()   // always disable network cache for fresh data
    .build()
    .getObjectObservable<PhotoResponse>(PhotoResponse::class.java)
```

- **Cache policy:** `doNotCacheResponse()` is called on all requests — responses are not cached by the networking layer.
- **No request body** for current GET calls.
- **No auth headers** — JSONPlaceholder is unauthenticated. Adding authentication should be done by configuring an OkHttp interceptor in `RestApiModule`.

## Response Format

Responses are typed via `getObjectObservable<T>(T::class.java)`, where `T` is a response class from `android_model`.

Current response types are plain `ArrayList` subclasses:

```kotlin
// android_model/src/main/java/com/gl/kev/model/io/Responses.kt
class PhotoResponse : ArrayList<Photo>()
class TodoResponse : ArrayList<Todo>()
```

The API returns JSON arrays directly (not wrapped in an object). New endpoints that return a JSON object wrapper should define a dedicated response data class in `:android_model`.

## Interface Definition

Every new API method must be declared in the `ApiHelper` interface before being implemented:

```kotlin
// ApiHelper.kt
interface ApiHelper {
    fun doGetPhotos(): Observable<PhotoResponse>
    fun doGetTodos(): Observable<TodoResponse>
    // Add new methods here
}
```

Method naming convention: `do<Verb><Resource>()` (e.g., `doGetUsers()`, `doPostComment()`).

## DataManager Integration

API calls are exposed to ViewModels through `DataManager`, not `ApiHelper` directly. `AppDataManager` subscribes on `io()` and observes on `ui()` using `SchedulerProvider`:

```kotlin
// AppDataManager.kt
fun getTodos(response: Consumer<TodoResponse>, failure: Consumer<Throwable>) {
    getCompositeDisposable().add(
        mApiHelper.doGetTodos()
            .subscribeOn(mSchedulerProvider.io())
            .observeOn(mSchedulerProvider.ui())
            .subscribe(response, failure)
    )
}
```

Rules:
- Always subscribe on `mSchedulerProvider.io()` and observe on `mSchedulerProvider.ui()`.
- Always add to `getCompositeDisposable()` to prevent leaks.
- Define both a success `Consumer<T>` and failure `Consumer<Throwable>` — never silently swallow errors.

## Error Handling

Error handling in ViewModels currently uses inline `Consumer<Throwable>` lambdas with `Log.e()`:

```kotlin
mDataManager.getPhotos(
    Consumer { Log.e("App", "I retrieved ${it.size} photos") },
    Consumer { Log.e("App", it.message, it) }
)
```

For production-ready error handling:
- Log errors at the ViewModel level.
- Expose error state to the UI via `LiveData<String>` or similar observable.
- Never show raw exception messages to end users.

## Base URL Configuration

The base URL is a `BuildConfig` field set in `app/build.gradle`:

```groovy
buildConfigField("String", "BASE_ENDPOINT_URL", "\"https://jsonplaceholder.typicode.com\"")
```

To add per-environment URLs (staging, production), define them per `buildType` or `productFlavor` in `app/build.gradle` rather than hardcoding them.

## Adding a New API Endpoint

1. Add the URL constant to `ApiEndPoint.kt`.
2. Add the method signature to `ApiHelper.kt`.
3. Implement the method in `AppApiHelper.kt` using the `Rx2AndroidNetworking` builder.
4. Add response/request model classes to `:android_model` if needed.
5. Expose the call in `DataManager.kt` and implement in `AppDataManager.kt` with proper scheduler threading.
6. Inject via `DataManager` in the relevant `ViewModel`.
