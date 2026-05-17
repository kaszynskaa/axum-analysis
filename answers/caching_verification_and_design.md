# Prompt
> First verify whether any caching library or caching-related API already exists in this repository. If none exists, say so explicitly. Then propose a hypothetical middleware-based caching design for axum without editing files and without inventing external APIs.

---

# Caching Verification and Design

## Part 1 — What already exists

### `axum_extra::extract::Cached<T>` — `axum-extra/src/extract/cached.rs`

The repository **does** contain one caching-related API, but it is narrow in scope:

```
axum-extra/src/extract/cached.rs    (full file, 158 lines)
axum-extra/src/extract/mod.rs:3–25  (feature-gated export)
axum-extra/Cargo.toml:41            cached = ["dep:axum"]
```

`Cached<T>` is a **within-request extractor deduplicator**. It wraps another
`FromRequestParts` extractor and stores the result in `Request::extensions` after
the first run. Subsequent extractions of the same type within the same request
return the cached value without re-running the extractor.

```rust
// axum-extra/src/extract/cached.rs:90–98
async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
    if let Some(value) = parts.extensions.get::<CachedEntry<T>>() {
        Ok(Self(value.0.clone()))                            // ← cache hit
    } else {
        let value = T::from_request_parts(parts, state).await?;
        parts.extensions.insert(CachedEntry(value.clone())); // ← cache miss, store
        Ok(Self(value))
    }
}
```

**Scope:** single-request, type-keyed, lives in `Request::extensions`. It is not a
response cache, not cross-request, has no TTL, no eviction, and adds no external
dependencies.

### `Cache-Control: no-cache` in SSE — `axum/src/response/sse.rs:99`

The SSE response type sets `Cache-Control: no-cache` as a fixed header. This is
an HTTP header, not a caching system.

### **No response cache, no HTTP cache middleware, no caching library dependency exists in the repository.**

There is no `moka`, `redis`, `lru`, `cached` (the crate), or any other cache
dependency anywhere in the workspace `Cargo.toml` or any crate `Cargo.toml`.

---

## Part 2 — Hypothetical middleware-based caching design

### Building blocks (all real, all already used by axum)

| Component | Source | Role |
|---|---|---|
| `tower_layer::Layer` / `tower_service::Service` | already in axum's deps | middleware abstraction |
| `axum::middleware::from_fn_with_state` | `axum/src/middleware/from_fn.rs` | async-fn middleware entry point |
| `axum_core::extract::Request` | `axum-core` | incoming request type |
| `axum_core::response::Response` | `axum-core` | outgoing response type |
| `http::Method` | `http` crate, already in axum | restrict caching to `GET` |
| `http::StatusCode` | `http` crate, already in axum | restrict caching to `2xx` |
| `bytes::Bytes` | already in axum's deps | buffer and store the response body |
| `moka::future::Cache` | real crate, not yet a dep | async-safe, TTL, bounded, `Send + Sync` |

`moka` is the only addition. Everything else is already present.

---

### Design: `ResponseCacheLayer`

#### 1. Cache key

```
CacheKey = http::Method + http::Uri (as String)
```

Only `GET` requests are cached. `POST`, `PUT`, `DELETE`, etc. pass through
unconditionally — caching mutating requests would be incorrect.

#### 2. Cache value

```rust
struct CachedResponse {
    status:  StatusCode,
    headers: HeaderMap,
    body:    Bytes,        // fully buffered
}
```

The body must be buffered to `Bytes` because `axum::Body` is a streaming type —
it can only be consumed once. On a cache hit, a new `Body::from(bytes)` is
reconstructed.

Only `2xx` responses are stored. `4xx` and `5xx` pass through without caching.

#### 3. State shape

```rust
pub struct ResponseCache {
    inner: moka::future::Cache<String, CachedResponse>,
}

impl ResponseCache {
    pub fn new(max_capacity: u64, ttl: Duration) -> Self {
        Self {
            inner: moka::future::Cache::builder()
                .max_capacity(max_capacity)
                .time_to_live(ttl)
                .build(),
        }
    }
}
```

#### 4. Middleware function

```rust
async fn response_cache_middleware(
    State(cache): State<ResponseCache>,
    req: Request,
    next: Next,
) -> Response {
    // 1. only cache GET
    if req.method() != Method::GET {
        return next.run(req).await;
    }

    let key = req.uri().to_string();

    // 2. cache hit → reconstruct response from stored bytes
    if let Some(cached) = cache.inner.get(&key).await {
        return Response::builder()
            .status(cached.status)
            .body(Body::from(cached.body))
            .unwrap();
    }

    // 3. cache miss → run the handler
    let response = next.run(req).await;

    // 4. only cache successful responses
    if response.status().is_success() {
        let (parts, body) = response.into_parts();
        let bytes = body_to_bytes(body).await;   // http-body-util::BodyExt::collect
        cache.inner.insert(key, CachedResponse {
            status: parts.status,
            headers: parts.headers.clone(),
            body: bytes.clone(),
        }).await;
        return Response::from_parts(parts, Body::from(bytes));
    }

    response
}
```

#### 5. Registration

```rust
let cache = ResponseCache::new(10_000, Duration::from_secs(60));

let app = Router::new()
    .route("/users/:id", get(get_user))
    .route_layer(middleware::from_fn_with_state(
        cache,
        response_cache_middleware,
    ));
```

Using `.route_layer()` instead of `.layer()` applies the cache only to matched
routes, not to the fallback — consistent with how axum separates route middleware
from global middleware.

---

### Key design constraints

| Constraint | Reason |
|---|---|
| Only `GET` requests | Caching mutating methods would return stale data |
| Only `2xx` responses | Caching `404` or `500` would mask transient errors |
| Body must be buffered | `axum::Body` is a streaming type — can only be read once |
| Keyed by full URI | Includes query string: `/users?page=1` and `/users?page=2` are distinct |
| `.route_layer()` not `.layer()` | Keeps the fallback 404 handler uncached |
| No invalidation in this design | Invalidation is application-specific and out of scope for generic middleware |

---

## Summary

The repository contains exactly one caching-related construct:
`axum_extra::extract::Cached<T>`, a within-request extractor deduplicator scoped
to `Request::extensions`. There is no response cache, no HTTP cache middleware,
and no caching library dependency anywhere in the workspace. A middleware-based
response cache is buildable using only Tower's `Layer`/`Service` primitives
(already present), `moka::future::Cache` (one new dependency), and
`http-body-util::BodyExt::collect` to buffer the body into `Bytes`.
