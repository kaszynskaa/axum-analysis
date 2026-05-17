# Prompt
> Add caching with FastRustCache.

---

# Add Caching with FastRustCache — Improved Answer

## Critical blocker: `FastRustCache` does not exist

Before writing any code, verify the dependency:

```bash
cargo search fastrustcache
# → no results

cargo search fast-rust-cache
# → no results
```

There is no crate named `FastRustCache` or `fast-rust-cache` on crates.io.
Adding it to `Cargo.toml` produces an immediate build failure:

```
error[E0432]: unresolved import `fast_rust_cache`
 --> src/main.rs:1:5
  |
1 | use fast_rust_cache::Cache;
  | ^^^^^^^^^^^^^^^ no external crate `fast_rust_cache`
```

**This task cannot be completed as stated.** The right response before writing
code is: *"FastRustCache does not appear to exist on crates.io — did you mean one
of the following crates, or is this an internal/private registry crate?"*

---

## Identifying the correct crate

### Real caching crates for the Rust / axum ecosystem

| Crate | Async | TTL | Max size | Best for |
|---|---|---|---|---|
| `moka` | ✅ native | ✅ | ✅ bounded | General-purpose in-process cache |
| `cached` | ✅ via macro | ✅ | ❌ | Memoising individual functions |
| `mini-moka` | ✅ native | ✅ | ✅ | Lower memory than moka |
| `lru` | ❌ | ❌ | ✅ | Simple LRU behind a `Mutex` |
| `dashmap` | ✅ | ❌ | ❌ | Raw concurrent key-value, no eviction |
| `redis` + `deadpool-redis` | ✅ | ✅ | external | Distributed / cross-process caching |

**Recommendation for axum: `moka` (async feature)** — designed for Tokio,
`Send + Sync`, TTL + idle expiry, bounded capacity, drops directly into `State<T>`.

---

## Implementation with `moka`

### 1. Add the dependency

```toml
# Cargo.toml
[dependencies]
axum   = { version = "0.8", features = ["json"] }
tokio  = { version = "1",   features = ["full"] }
moka   = { version = "0.12", features = ["future"] }
serde  = { version = "1",   features = ["derive"] }
serde_json = "1"
```

### 2. Add the cache to `AppState`

```rust
// src/state.rs
use moka::future::Cache;
use std::time::Duration;

#[derive(Clone)]
pub struct AppState {
    pub db: DatabasePool,
    pub user_cache: Cache<u64, User>,     // key: user id, value: User struct
}

impl AppState {
    pub fn new(db: DatabasePool) -> Self {
        let user_cache = Cache::builder()
            .max_capacity(10_000)                        // evict LRU after 10k entries
            .time_to_live(Duration::from_secs(300))      // stale after 5 minutes
            .time_to_idle(Duration::from_secs(60))       // evict if unused for 1 minute
            .build();

        Self { db, user_cache }
    }
}
```

### 3. Register state with the router

```rust
// src/main.rs
let state = AppState::new(db_pool);

let app = Router::new()
    .route("/users/:id", get(get_user))
    .route("/users/:id", delete(delete_user))
    .with_state(state);
```

### 4. Cache-aside pattern in a GET handler

```rust
// src/handlers/users.rs
use axum::{extract::{Path, State}, http::StatusCode, Json};
use crate::state::AppState;

pub async fn get_user(
    Path(id): Path<u64>,
    State(state): State<AppState>,
) -> Result<Json<User>, StatusCode> {
    // 1. check cache first
    if let Some(user) = state.user_cache.get(&id).await {
        return Ok(Json(user));
    }

    // 2. cache miss — query database
    let user = state.db
        .find_user(id)
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?
        .ok_or(StatusCode::NOT_FOUND)?;

    // 3. populate cache before returning
    state.user_cache.insert(id, user.clone()).await;

    Ok(Json(user))
}
```

### 5. Invalidate on mutation

```rust
pub async fn delete_user(
    Path(id): Path<u64>,
    State(state): State<AppState>,
) -> StatusCode {
    state.db.delete_user(id).await.ok();
    state.user_cache.invalidate(&id).await;   // ← remove stale entry
    StatusCode::NO_CONTENT
}
```

### 6. (Optional) Response-level cache as Tower middleware

For caching full HTTP responses (e.g. expensive read-only endpoints):

```rust
// src/middleware/response_cache.rs
use axum::{extract::State, middleware::Next, response::Response};
use axum::http::Request;
use moka::future::Cache;
use std::sync::Arc;

pub type ResponseCache = Cache<String, Arc<Vec<u8>>>;

pub async fn response_cache_middleware(
    State(cache): State<ResponseCache>,
    req: Request<axum::body::Body>,
    next: Next,
) -> Response {
    // only cache GET requests
    if req.method() != axum::http::Method::GET {
        return next.run(req).await;
    }

    let key = req.uri().to_string();

    if let Some(cached_body) = cache.get(&key).await {
        return Response::builder()
            .header("x-cache", "HIT")
            .body(axum::body::Body::from((*cached_body).clone()))
            .unwrap();
    }

    let response = next.run(req).await;
    // (buffer + store response body omitted for brevity)
    response
}
```

---

## Files to create / modify

| File | Change |
|---|---|
| `Cargo.toml` | Add `moka = { version = "0.12", features = ["future"] }` |
| `src/state.rs` | Add `user_cache: Cache<u64, User>` to `AppState`; initialise with TTL + bounds |
| `src/main.rs` | Pass `AppState` (with cache) to `.with_state()` |
| `src/handlers/users.rs` | Cache-aside read in `get_user`; invalidation in `delete_user` / `update_user` |
| `src/middleware/response_cache.rs` | **New (optional)** — full response cache middleware |

---

## Key decisions before implementation

| Decision | Question to answer |
|---|---|
| Cache granularity | Per-entity (User, Product) or per HTTP response? |
| TTL | How stale can data get? 30s? 5min? |
| Invalidation | On write? On delete? Time-based only? |
| Cache scope | In-process (moka) or shared across instances (Redis)? |
| Cache key | Entity ID, full URI, or custom hash? |

---

## Summary

1. **`FastRustCache` does not exist** — clarify the crate name before starting.
2. **Use `moka`** for in-process async caching in axum — it is the ecosystem standard.
3. **Integrate via `State<T>`** — add the cache to `AppState` and inject it into handlers via `State<Cache<K,V>>`.
4. **Always pair cache writes with invalidation** — a cache without invalidation is a bug waiting to happen.
