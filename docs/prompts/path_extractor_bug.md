# Prompt
> Search the axum repository for one concrete potential bug or risky behavior related to request routing or extractors. Return only: file path, line number, code evidence, explanation, severity, and how to verify it manually.

---

# Potential Bug: `Path<T>` in a Fallback Handler Silently Returns 500

## File paths and line numbers

| Location | Line | Role |
|---|---|---|
| `axum/src/extract/rejection.rs` | 50–56 | Defines `MissingPathParams` → `500 Internal Server Error` |
| `axum/src/extract/path/mod.rs` | 167–179 | `Path<T>::from_request_parts` — fires the rejection |
| `axum/src/routing/url_params.rs` | 12 | `insert_url_params` — the only place `UrlParams` is ever written |
| `axum/src/routing/path_router.rs` | 353 | Where `insert_url_params` is called (only on a matched route) |

---

## Code evidence

**`axum/src/extract/rejection.rs:50–56`**
```rust
define_rejection! {
    #[status = INTERNAL_SERVER_ERROR]
    #[body = "No paths parameters found for matched route"]
    pub struct MissingPathParams;
}
```

**`axum/src/extract/path/mod.rs:167–179`**
```rust
match parts.extensions.get::<UrlParams>() {
    Some(UrlParams::Params(params)) => Ok(params),
    Some(UrlParams::InvalidUtf8InPathParam { key }) => { … }
    None => Err(MissingPathParams.into()),   // ← fires a 500
}
```

**`axum/src/routing/url_params.rs:12`**
```rust
pub(super) fn insert_url_params(extensions: &mut Extensions, params: &Params<'_, '_>) { … }
```

`UrlParams` is inserted **only** by `PathRouter::call_with_state` at `path_router.rs:353`,
which runs only when the matchit radix tree finds a match. It is never set for fallback
handlers, `route_service` routes, or any middleware that reconstructs the `Request`.

---

## Explanation

`Path<T>` reads captured URL parameters from `Request::extensions` under the key
`UrlParams`. This entry is written by the path-matching step in `PathRouter`. If
`Path<T>` is used in a context where path matching never ran — most commonly a
**`.fallback()` handler** — the extension is absent, the `None` branch fires, and
the extractor returns `MissingPathParams`, which maps to **`500 Internal Server Error`**.

The footgun: the compiler accepts this code without warning. There is no trait bound,
type-state, or lint to signal that `Path<T>` in a fallback is meaningless. The error
is invisible until a live request hits the handler.

The library's own doc comment on `MissingPathParams` acknowledges this:
> *"This is commonly caused by extracting `Request<_>`. `Path` must be extracted first."*

---

## Severity

**Medium**

- Cannot cause data loss or security issues.
- Produces a misleading `500 Internal Server Error` instead of a more appropriate
  `404` or `400`, making debugging harder.
- Invisible at compile time and at application startup — only discovered at runtime
  when a request hits the affected handler.
- No compiler warning, no `#[must_use]`, no panic — just a silent wrong response.

---

## How to verify manually

```rust
use axum::{Router, routing::get, extract::Path};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/users/:id", get(|| async { "ok" }))
        .fallback(|Path(id): Path<String>| async move {
            format!("fallback for {id}")
        });

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

Send a request that does not match `/users/:id`:

```bash
curl -i http://localhost:3000/anything-else
# HTTP/1.1 500 Internal Server Error
# body: No paths parameters found for matched route
```

The `500` confirms the bug. The correct fix is to remove `Path<T>` from the fallback
handler — fallback handlers receive no captured path parameters by definition.
