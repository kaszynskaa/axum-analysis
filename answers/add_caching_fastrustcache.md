# Add Caching with FastRustCache

## Problem with this task

**`FastRustCache` does not exist.** There is no published crate with that name on
crates.io. Implementing a dependency on a non-existent crate would break the build
immediately.

```
error[E0432]: unresolved import `fast_rust_cache`
 --> src/main.rs:1:5
  |
1 | use fast_rust_cache::Cache;
  | ^^^^^^^^^^^^^^^^^^^^^^^ no external crate `fast_rust_cache`
```

Before proceeding, the correct crate must be identified.

---

## Real caching crates for Rust / axum

| Crate | Type | Best for |
|---|---|---|
| `moka` | Async-native, LRU/LFU, TTL, thread-safe | General-purpose in-process cache |
| `cached` | Proc-macro (`#[cached]`) | Memoising individual functions |
| `mini-moka` | Lightweight version of moka | Lower memory / simpler use cases |
| `lru` | Simple LRU, no async | Single-threaded or `Mutex`-wrapped |
| `dashmap` | Concurrent `HashMap` | Raw key-value store without eviction |
| `redis` + `deadpool-redis` | External Redis | Distributed / cross-process caching |

**Recommended for axum: `moka`** — async-native, TTL support, bounded size,
`Send + Sync`, drops straight into `State<T>`.

---

## How to add `moka` caching to axum

### 1. Add the dependency

```toml
# Cargo.toml
[dependencies]
moka = { version = "0.12", features = ["future"] }
```

### 2. Define the cache as part of app state

```rust
// src/state.rs
use moka::future::Cache;

#[derive(Clone)]
pub struct AppState {
    pub db: DatabasePool,
    pub cache: Cache<String, Vec<u8>>,   // key: String, value: serialised bytes
}

impl AppState {
    pub fn new(db: DatabasePool) -> Self {
        let cache = Cache::builder()
            .max_capacity(10_000)
            .time_to_live(Duration::from_secs(60))
            .time_to_idle(Duration::from_secs(30))
            .build();
        Self { db, cache }
    }
}
```

### 3. Register state with the router

```rust
// src/main.rs
let state = AppState::new(db_pool);

let app = Router::new()
    .route("/users/:id", get(get_user))
    .with_state(state);
```

### 4. Use the cache in a handler

```rust
// src/handlers/users.rs
use axum::{extract::{Path, State}, Json};
use crate::state::AppState;

pub async fn get_user(
    Path(id): Path<String>,
    State(state): State<AppState>,
) -> Json<User> {
    // cache-aside pattern
    if let Some(bytes) = state.cache.get(&id).await {
        let user: User = serde_json::from_slice(&bytes).unwrap();
        return Json(user);
    }

    let user = state.db.find_user(&id).await.unwrap();
    let bytes = serde_json::to_vec(&user).unwrap();
    state.cache.insert(id, bytes).await;

    Json(user)
}
```

### 5. (Optional) Cache as Tower middleware

For response-level caching (cache the full HTTP response):

```rust
// src/middleware/cache.rs
use axum::{middleware::Next, extract::{Request, State}};
use axum::response::Response;

pub async fn cache_middleware(
    State(state): State<AppState>,
    req: Request,
    next: Next,
) -> Response {
    let key = req.uri().to_string();

    if let Some(cached) = state.cache.get(&key).await {
        return cached;   // Cache<String, Response> — Response must be Clone
    }

    let response = next.run(req).await;
    state.cache.insert(key, response.clone()).await;
    response
}
```

Applied to the router:

```rust
use axum::middleware;

let app = Router::new()
    .route("/users/:id", get(get_user))
    .layer(middleware::from_fn_with_state(state.clone(), cache_middleware))
    .with_state(state);
```

---

## Files to create / modify

| File | Change |
|---|---|
| `Cargo.toml` | Add `moka` dependency |
| `src/state.rs` | Add `Cache` field to `AppState` |
| `src/main.rs` | Initialise cache, pass via `.with_state()` |
| `src/handlers/*.rs` | Add cache-aside logic per handler |
| `src/middleware/cache.rs` | **New** — optional response-level cache middleware |

---

## Recommendation

1. **Clarify what `FastRustCache` refers to** — if it is an internal/proprietary crate,
   provide the crate name, registry URL, and API docs before implementation begins.
2. **If it is a typo or placeholder**, use `moka` — it is the standard async-safe
   cache for the Tokio ecosystem and integrates cleanly with axum's `State<T>` pattern.
3. **Do not add caching without a cache invalidation strategy** — decide on TTL values,
   maximum capacity, and how stale entries are handled before writing any code.
