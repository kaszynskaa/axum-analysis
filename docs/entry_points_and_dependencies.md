# axum — Entry Points and External Dependencies

## Main entry points

### 1. `axum::serve()` — start the HTTP server

**File:** `axum/src/serve/mod.rs:103`  
**Re-exported from:** `axum/src/lib.rs`

```rust
pub fn serve<L, M, S, B>(listener: L, make_service: M) -> Serve<…>
```

The only way to bind a socket and start serving requests. Accepts a
`tokio::net::TcpListener` (or any type implementing the `Listener` trait) and
any Tower `MakeService` — in practice always a `Router<()>`.

```rust
let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
axum::serve(listener, app).await?;
```

Internally drives `hyper_util::server::conn::auto::Builder` (HTTP/1 + HTTP/2)
and spawns one Tokio task per accepted connection.

---

### 2. `Router<S>` — define routes

**File:** `axum/src/routing/mod.rs:86`  
**Re-exported from:** `axum/src/lib.rs`

```rust
pub struct Router<S = ()> { inner: Arc<RouterInner<S>> }
```

The primary application-builder type. Key methods:

| Method | File : line | Purpose |
|---|---|---|
| `Router::new()` | `routing/mod.rs:160` | Create empty router |
| `.route(path, method_router)` | `routing/mod.rs:192` | Register a path |
| `.nest(path, router)` | `routing/mod.rs:220` | Mount a sub-router |
| `.merge(router)` | `routing/mod.rs:258` | Merge two routers |
| `.layer(layer)` | `routing/mod.rs:295` | Apply Tower middleware to all routes |
| `.route_layer(layer)` | `routing/mod.rs:313` | Apply middleware only to matched routes |
| `.with_state(state)` | `routing/mod.rs:436` | Seal state → produces `Router<()>` |
| `.fallback(handler)` | `routing/mod.rs:358` | Custom 404 handler |

`Router<S>` is generic over missing state `S`. Only `Router<()>` can be passed
to `serve()`. The type-state enforces at compile time that all state is provided
before the server starts.

---

### 3. Method routing helpers — attach handlers

**File:** `axum/src/routing/method_routing.rs`  
**Re-exported from:** `axum/src/routing/mod.rs:45`

```rust
pub use method_routing::{get, post, put, delete, patch, head, options, trace, any, on};
```

Each function (e.g. `get(handler)`) returns a `MethodRouter` that is passed to
`Router::route`. Handlers are any async function whose arguments implement
`FromRequest`/`FromRequestParts` and whose return value implements `IntoResponse`.

---

### 4. `Handler` trait

**File:** `axum/src/handler/mod.rs:100`

```rust
pub trait Handler<T, S>: Clone + Send + Sized + 'static {
    type Future: Future<Output = Response> + Send + 'static;
    fn call(self, req: Request, state: S) -> Self::Future;
}
```

Blanket-implemented via the `impl_handler!` macro for all async functions of
arity 0–16 whose argument types implement `FromRequestParts` (all but the last)
and `FromRequest` (the last).

---

### 5. `FromRequest` / `FromRequestParts` — extractor traits

**File:** `axum-core/src/extract/mod.rs`  
**Re-exported from:** `axum/src/lib.rs`

```rust
pub trait FromRequestParts<S>: Sized {
    type Rejection: IntoResponse;
    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection>;
}

pub trait FromRequest<S, M = private::ViaRequest>: Sized {
    type Rejection: IntoResponse;
    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection>;
}
```

Implement these traits to create custom extractors. `FromRequestParts` cannot
consume the body; `FromRequest` may.

---

### 6. `IntoResponse` — response trait

**File:** `axum-core/src/response/mod.rs`  
**Re-exported from:** `axum/src/lib.rs`

```rust
pub trait IntoResponse {
    fn into_response(self) -> Response;
}
```

Implemented for all handler return types: tuples of `(StatusCode, headers, body)`,
`Json<T>`, `Html<T>`, `String`, `&'static str`, `Bytes`, `StatusCode`, `Result<T, E>`, etc.

---

## External dependencies

### Core runtime and HTTP

| Crate | Version | Role |
|---|---|---|
| `tokio` | 1.44 | Async runtime — task scheduling, `TcpListener`, timers, `sync::watch` |
| `hyper` | 1.4.0 | HTTP/1.1 and HTTP/2 protocol implementation |
| `hyper-util` | 0.1.4 | `TowerToHyperService`, `auto::Builder`, `TokioIo`, `TokioExecutor` |
| `tower` | 0.5.2 | `Service` + `Layer` abstractions; `ServiceExt` utilities |
| `tower-service` | 0.3 | `Service` trait definition |
| `tower-layer` | 0.3.2 | `Layer` trait definition |
| `tower-http` | 0.6.8 | Ready-made HTTP middleware: `Trace`, `Cors`, `Compression`, `Auth`, `Limit`, … |

### HTTP types

| Crate | Version | Role |
|---|---|---|
| `http` | 1.0.0 | `Request`, `Response`, `Method`, `StatusCode`, `HeaderMap`, `Uri` types |
| `http-body` | 1.0.0 | `Body` trait |
| `http-body-util` | 0.1.0 | `BodyExt`, `Limited`, `Empty`, `Full` body utilities |
| `bytes` | 1.7 | Zero-copy `Bytes` and `BytesMut` byte buffers |

### Routing

| Crate | Version | Role |
|---|---|---|
| `matchit` | 0.9.2 | Fast radix-tree router used by `PathRouter` for path-parameter matching |

### Serialisation

| Crate | Version | Role |
|---|---|---|
| `serde` | 1.0.221 | Serialisation framework (traits only) |
| `serde_json` | 1.0.29 | JSON — used by `Json<T>` extractor/response (`json` feature) |
| `serde_html_form` | 0.4.0 | HTML form decoding (`form` feature) |
| `form_urlencoded` | 1.1.0 | URL-encoded form encoding (`form` feature) |
| `serde_path_to_error` | — | Wraps serde errors with the field path that failed |
| `mime` | 0.3.16 | MIME type parsing for `Content-Type` validation |

### Async utilities

| Crate | Version | Role |
|---|---|---|
| `futures-core` | 0.3 | `Stream` and `Future` primitives |
| `futures-util` | 0.3 | `FutureExt`, `StreamExt` combinators |
| `pin-project-lite` | 0.2.7 | Safe `Pin` projection macros |
| `sync_wrapper` | 1.0.0 | Makes `!Send` types usable in `Send` contexts via wrapping |

### Optional / feature-gated

| Crate | Feature flag | Role |
|---|---|---|
| `tokio-tungstenite` | `ws` | WebSocket upgrade handling |
| `sha1` + `base64` | `ws` | WebSocket handshake hashing/encoding |
| `multer` | `multipart` | Multipart form-data body parsing |
| `tracing` | `tracing` | Structured logging in extractors and serve |
| `percent-encoding` | — | URL percent-encode/decode for path params |
| `memchr` | — | Fast byte searching in body parsing |
| `itoa` | — | Fast integer-to-ASCII conversion |

### Proc-macro crate (`axum-macros`) only

| Crate | Version | Role |
|---|---|---|
| `syn` | 2.0 | Rust AST parsing |
| `quote` | 1.0 | Token-stream code generation |
| `proc-macro2` | 1.0 | Macro token stream API |

---

## Feature flags summary

| Flag | What it enables |
|---|---|
| `http1` | HTTP/1.1 via hyper (default) |
| `http2` | HTTP/2 via hyper |
| `json` | `Json<T>` extractor and response |
| `form` | `Form<T>` extractor |
| `query` | `Query<T>` extractor |
| `multipart` | `Multipart` extractor |
| `ws` | WebSocket upgrade support |
| `tokio` | `axum::serve()`, `ConnectInfo`, `TcpListener` integration (default) |
| `tracing` | Tracing instrumentation |
| `matched-path` | `MatchedPath` extractor |
| `original-uri` | `OriginalUri` extractor |
| `macros` | Re-exports from `axum-macros` |
