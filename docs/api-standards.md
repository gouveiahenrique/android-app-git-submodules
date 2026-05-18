# API Standards

**Last updated:** 2026-05-18

## Overview

This app consumes a public REST API (`https://jsonplaceholder.typicode.com`) as a read-only client. There is no server-side code in this repository. All API interaction is handled through the `ApiHelper` interface in `:app` using the Rx2AndroidNetworking library.

## REST Conventions

### HTTP Methods Used

- **GET** — Only `GET` calls are currently implemented (todos, photos).

### URL Patterns

```
GET  https://jsonplaceholder.typicode.com/todos    # Fetch all todos
GET  https://jsonplaceholder.typicode.com/photos   # Fetch all photos
```

All endpoint constants are defined in `ApiEndPoint.kt` using the `BuildConfig.BASE_ENDPOINT_URL` field set at build time:

```kotlin
// app/src/main/java/com/gl/kev/app/data/api/ApiEndPoint.kt
object ApiEndPoint {
    const val API_URL_TODOS  = "${BuildConfig.BASE_ENDPOINT_URL}/todos"
    const val API_URL_PHOTOS = "${BuildConfig.BASE_ENDPOINT_URL}/photos"
}
```

### Status Codes (External API)

The external API follows standard REST conventions. The app currently does not handle specific status codes explicitly — failures are propagated as `Throwable` via the RxJava `Consumer<Throwable>` error path.

## Request Format

Requests are plain `GET` calls with no request body or custom headers. `Rx2AndroidNetworking` handles the OkHttp transport layer:

```kotlin
Rx2AndroidNetworking.get(ApiEndPoint.API_URL_PHOTOS)
    .doNotCacheResponse()
    .build()
    .getObjectObservable<PhotoResponse>(PhotoResponse::class.java)
```

- **Caching:** `.doNotCacheResponse()` — all responses bypass cache.
- **Authentication:** None (public API, no auth token required).

## Response Format

### Photos Response

The API returns a JSON array of photo objects. The app deserializes this using Gson via the `Rx2AndroidNetworking` library into `PhotoResponse` (a typed `ArrayList<Photo>`):

```json
[
  {
    "albumId": 1,
    "id": 1,
    "title": "accusamus beatae ad facilis cum similique qui sunt",
    "url": "https://via.placeholder.com/600/92c952",
    "thumbnailUrl": "https://via.placeholder.com/150/92c952"
  }
]
```

Mapped to:
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

### Todos Response

```json
[
  {
    "userId": 1,
    "id": 1,
    "title": "delectus aut autem",
    "completed": false
  }
]
```

Mapped to:
```kotlin
@Entity(tableName = "todo")
data class Todo(
    @PrimaryKey val id: Int,
    val userId: Int,
    var title: String,
    var completed: Boolean
)
```

## Error Handling

All API calls return `Observable<T>`. Errors are surfaced via the RxJava error consumer passed at the call site:

```kotlin
mDataManager.getPhotos(
    Consumer { response -> /* success */ },
    Consumer { error ->
        Log.e("App", error.message, error)
    }
)
```

`AppDataManager` subscribes on the IO scheduler and observes on the UI thread:

```kotlin
mApiHelper.doGetPhotos()
    .subscribeOn(mSchedulerProvider.io())
    .observeOn(mSchedulerProvider.ui())
    .subscribe(response, failure)
```

`CompositeDisposable` in `BaseDataManager` manages all subscriptions; call `dispose()` in `onCleared()` to prevent memory leaks.

## Adding New Endpoints

1. Add the URL constant to `ApiEndPoint.kt`.
2. Add the method signature to `ApiHelper.kt` (interface).
3. Implement it in `AppApiHelper.kt` using `Rx2AndroidNetworking`.
4. Add the corresponding domain method to `DataManager.kt` (interface).
5. Implement it in `AppDataManager.kt`, scheduling on `mSchedulerProvider.io()` / `mSchedulerProvider.ui()`.
6. If a new entity is involved, define the `data class` in `:android_model` and add a DAO in `:app`.

## Versioning

The current API target (`jsonplaceholder.typicode.com`) is not versioned. For future production APIs:

- Use URL path versioning: `/api/v1/resource`
- Bump `BASE_ENDPOINT_URL` in `build.gradle` `buildConfigField` for environment switching (dev/staging/prod).

## Authentication (Future)

Currently no authentication is required. If an authenticated API is added:

- Store tokens securely (Android Keystore or EncryptedSharedPreferences).
- Inject the token via an OkHttp `Interceptor` configured inside `RestApiModule`.
- Do **not** hardcode tokens in `BuildConfig` fields or source code.
