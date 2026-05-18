# API Standards

**Last updated:** 2026-05-18

## Overview

This project consumes a third-party REST API (JSONPlaceholder) — it does not expose its own server-side API. All conventions described here apply to **how the app calls the API** and how responses are mapped to local models.

## REST Conventions

### HTTP Methods Used

- **GET** — All current API calls are read-only GETs; no POST, PUT, PATCH, or DELETE operations are implemented

### URL Patterns

Endpoints are defined as constants in `ApiEndPoint.kt` (app/src/main/java/com/gl/kev/app/data/api/ApiEndPoint.kt):

```kotlin
object companion {
    const val API_URL_TODOS  = "${BuildConfig.BASE_ENDPOINT_URL}/todos"   // GET /todos
    const val API_URL_PHOTOS = "${BuildConfig.BASE_ENDPOINT_URL}/photos"  // GET /photos
}
```

- Base URL is injected at build time via `BuildConfig.BASE_ENDPOINT_URL` (default: `https://jsonplaceholder.typicode.com`)
- All endpoint constants live in `ApiEndPoint` — never hardcode URLs elsewhere

### Current Endpoints

| Method | Path | Response Type | Description |
|--------|------|---------------|-------------|
| GET | `/todos` | `TodoResponse` (`ArrayList<Todo>`) | Retrieve all todos |
| GET | `/photos` | `PhotoResponse` (`ArrayList<Photo>`) | Retrieve all photos |

## Request Format

### Headers

No custom headers are set currently. The `rx2-android-networking` library (wrapping OkHttp) sets standard defaults:
- `Content-Type: application/json` (inferred from response deserialization)
- No `Authorization` header — JSONPlaceholder is a public, unauthenticated API

### Caching

All requests use `.doNotCacheResponse()` — responses are never cached by the HTTP layer:

```kotlin
Rx2AndroidNetworking.get(ApiEndPoint.API_URL_PHOTOS)
    .doNotCacheResponse()
    .build()
    .getObjectObservable<PhotoResponse>(PhotoResponse::class.java)
```

## Response Format

### Deserialization

Responses are deserialized by Gson via `rx2-android-networking`'s `.getObjectObservable<T>(T::class.java)` pattern. Response types extend `ArrayList<Model>` directly:

```kotlin
class PhotoResponse : ArrayList<Photo>()
class TodoResponse : ArrayList<Todo>()
```

### Photo Object

```json
{
  "albumId": 1,
  "id": 1,
  "title": "accusamus beatae ad facilis cum similique qui sunt",
  "url": "https://via.placeholder.com/600/92c952",
  "thumbnailUrl": "https://via.placeholder.com/150/92c952"
}
```

Maps to:
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

### Todo Object

```json
{
  "userId": 1,
  "id": 1,
  "title": "delectus aut autem",
  "completed": false
}
```

Maps to:
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

Errors are propagated through the RxJava `Consumer<Throwable>` failure callback:

```kotlin
// In AppDataManager
mApiHelper.doGetTodos()
    .subscribeOn(mSchedulerProvider.io())
    .observeOn(mSchedulerProvider.ui())
    .subscribe(response, failure)

// In ViewModel call sites
mDataManager.getPhotos(
    Consumer { /* success: handle response */ },
    Consumer { Log.e("App", it.message, it) }  // failure: log throwable
)
```

- No structured error types or HTTP status code inspection is implemented currently
- All errors from network or Gson deserialization are passed as raw `Throwable` to the failure consumer
- Network errors (no connectivity, timeout) surface as `IOException` or OkHttp exceptions

## Authentication

Not applicable — JSONPlaceholder is a public mock API requiring no authentication.

When adding a secured API in the future:
- Inject auth tokens via an OkHttp `Interceptor` registered in `RestApiModule`
- Use `BuildConfig` fields or a `SharedPreferences`-backed token store

## Versioning

Not applicable to the current consumed API. JSONPlaceholder does not use versioned endpoints.

When exposing or consuming a versioned API:
- Prefer URL path versioning: `/api/v1/resource`
- Define version constant in `ApiEndPoint` companion object

## Adding New Endpoints

1. Add the URL constant to `ApiEndPoint`:
   ```kotlin
   const val API_URL_NEW_RESOURCE = "${BuildConfig.BASE_ENDPOINT_URL}/new-resource"
   ```
2. Add a corresponding response type in `android_model` (`io/Responses.kt` or a new file)
3. Declare the method on the `ApiHelper` interface returning `Observable<ResponseType>`
4. Implement it in `AppApiHelper` using `Rx2AndroidNetworking.get(...)` with `.doNotCacheResponse()`
5. Expose it through `DataManager` and implement in `AppDataManager`
6. Provide the scheduler chain: `.subscribeOn(mSchedulerProvider.io()).observeOn(mSchedulerProvider.ui())`
