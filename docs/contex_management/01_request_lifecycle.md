# Prompt
> Explain the complete lifecycle of an incoming HTTP request in axum, starting from
> Router::call() until the final HTTP response is produced. Include how route matching
> works, how URL parameters are inserted into Request::extensions, how extractors
> retrieve data, how MethodRouter dispatches by HTTP method, and how responses are
> converted into HTTP responses. Include relevant file paths and explain the interaction
> between the routing, extract, middleware, and response modules.

---

# Complete HTTP Request Lifecycle in axum

## Overview

The journey of a request through axum touches four modules in sequence: **routing**
(path and method matching), **extract** (pulling typed data out of the request),
**handler** (calling user code), and **response** (converting the return value into
bytes). Each module communicates with the next through well-defined trait boundaries
rather than direct function calls, which is what makes the system composable.

---

## Stage 1 — Entry point: `Router::call()`

**File:** `axum/src/routing/mod.rs:614`

```rust
impl<B> Service<Request<B>> for Router<()> {
    fn call(&mut self, req: Request<B>) -> Self::Future {
        let req = req.map(Body::new);      // wrap hyper Incoming → axum Body
        self.call_with_state(req, ())
    }
}
```

By the time `Router::call()` is reached, hyper has already parsed the raw TCP bytes
into an `http::Request<Incoming>`. axum wraps the body stream in its own `Body`
newtype (`axum/src/body/mod.rs`) and immediately forwards to `call_with_state`.

`call_with_state` (`mod.rs:452`) is the internal dispatch entry point shared by
all code paths (direct `Service::call`, `serve()`, and tests):

```rust
pub(crate) fn call_with_state(&self, req: Request, state: S) -> RouteFuture<Infallible> {
    match self.inner.path_router.call_with_state(req, state) {
        Ok(future) => return future,
        Err((req, state)) => { /* fall through to catch_all_fallback */ }
    }
    self.inner.catch_all_fallback.clone().call_with_state(req, state)
}
```

If `PathRouter` returns `Err`, the request missed every registered route and the
catch-all fallback (default 404 or user-supplied) handles it.

---

## Stage 2 — Path matching: `PathRouter` + `matchit`

**File:** `axum/src/routing/path_router.rs:325`

`PathRouter` owns two data structures: a `matchit::Router<RouteId>` radix tree
(wrapped in the private `Node` type at `path_router.rs:406`) and a
`Vec<Endpoint<S>>` indexed by `RouteId`. Registration happens at startup — every
`.route(path, handler)` call inserts the path into the radix tree and pushes the
endpoint into the vec, assigning `RouteId(vec.len())` as the key.

At request time:

```rust
// path_router.rs:340–370
let (mut parts, body) = req.into_parts();

match self.node.at(parts.uri.path()) {       // O(n) radix-tree lookup
    Ok(match_) => {
        let id = *match_.value;              // RouteId from the tree

        // (feature = "matched-path")
        set_matched_path_for_request(id, &self.node.route_id_to_path, &mut parts.extensions);

        url_params::insert_url_params(&mut parts.extensions, &match_.params);

        let endpoint = self.routes.get(id.0).expect("…");
        let req = Request::from_parts(parts, body);
        match endpoint {
            Endpoint::MethodRouter(mr) => Ok(mr.call_with_state(req, state)),
            Endpoint::Route(route)     => Ok(route.clone().call_owned(req)),
        }
    }
    Err(MatchError::NotFound) => Err((Request::from_parts(parts, body), state)),
}
```

The request is split into `(parts, body)` before the lookup so that extensions can
be mutated without moving the body. After the tree match, the request is reassembled
and forwarded.

---

## Stage 3 — URL parameters: `url_params::insert_url_params`

**File:** `axum/src/routing/url_params.rs:12`

```rust
pub(super) fn insert_url_params(extensions: &mut Extensions, params: &Params<'_, '_>) {
    // stores UrlParams::Params(vec) into Request::extensions
}
```

`matchit` returns the captured parameters as a `Params` iterator of
`(&str, &str)` pairs — for a route `/users/{id}` matched against `/users/42`,
this yields `("id", "42")`. `insert_url_params` encodes these as
`UrlParams::Params(Vec<(Arc<str>, PercentDecodedStr)>)` and stores the value
in `Request::extensions` under the type key `UrlParams`.

This is the **only place** `UrlParams` is ever written. No extractor, no middleware,
and no handler touches it directly — they all read it back later via
`Path<T>::from_request_parts`.

### How `Path<T>` reads them back

**File:** `axum/src/extract/path/mod.rs:167`

```rust
match parts.extensions.get::<UrlParams>() {
    Some(UrlParams::Params(params)) => Ok(params),
    Some(UrlParams::InvalidUtf8InPathParam { key }) => Err(…),
    None => Err(MissingPathParams.into()),   // 500 if called outside a matched route
}
```

`serde` then deserialises the `Vec<(Arc<str>, PercentDecodedStr)>` into `T` through
the custom `PathDeserializer` at `extract/path/de.rs`. This is why `Path<T>` requires
`T: serde::DeserializeOwned` — the deserialization is the mechanism.

---

## Stage 4 — Method dispatch: `MethodRouter::call_with_state`

**File:** `axum/src/routing/method_routing.rs:1167`

`MethodRouter` stores one `MethodEndpoint<S, E>` per HTTP verb (nine fields: `get`,
`head`, `post`, `put`, `delete`, `patch`, `options`, `trace`, `connect`) plus a
`fallback`. Dispatch is a cascade of inline `if` checks via the `call!` macro:

```rust
macro_rules! call {
    ($req:expr, $method_variant:ident, $svc:expr) => {
        if *req.method() == Method::$method_variant {
            match $svc {
                MethodEndpoint::None => {}
                MethodEndpoint::Route(route) => {
                    return route.clone().oneshot_inner_owned($req);
                }
                MethodEndpoint::BoxedHandler(handler) => {
                    let route = handler.clone().into_route(state);
                    return route.oneshot_inner_owned($req);
                }
            }
        }
    };
}

call!(req, HEAD, head);
call!(req, HEAD, get);     // HEAD falls through to GET if no HEAD handler
call!(req, GET,  get);
call!(req, POST, post);
// … remaining methods …
```

If no method matches, `fallback.call_with_state(req, state)` is called, which by
default returns `405 Method Not Allowed` with a populated `Allow` header built by
`append_allow_header` (`method_routing.rs:1225`).

When a match is found, `BoxedHandler::into_route(state)` bakes the state into
the handler, producing a `Route` (a type-erased Tower `Service`). The request is
then driven to completion with `.oneshot_inner_owned(req)`.

---

## Stage 5 — Extractor loop: `Handler::call`

**File:** `axum/src/handler/mod.rs:238`

The `impl_handler!` macro generates a `Handler` implementation for every function
arity from 0 to 16. For a handler with signature
`async fn(State<S>, Path<T>, Json<U>) -> impl IntoResponse`, the generated code
splits the request and runs extractors in order:

```rust
fn call(self, req: Request, state: S) -> Self::Future {
    let (mut parts, body) = req.into_parts();   // split head from body
    Box::pin(async move {

        // --- FromRequestParts extractors (non-body, run first) ---
        let state_val = match State::from_request_parts(&mut parts, &state).await {
            Ok(v)  => v,
            Err(r) => return r.into_response(),   // short-circuit with rejection
        };
        let path_val = match Path::from_request_parts(&mut parts, &state).await {
            Ok(v)  => v,
            Err(r) => return r.into_response(),
        };

        // --- FromRequest extractor (body-consuming, always last) ---
        let req = Request::from_parts(parts, body);
        let json_val = match Json::from_request(req, &state).await {
            Ok(v)  => v,
            Err(r) => return r.into_response(),
        };

        // --- invoke user function ---
        self(state_val, path_val, json_val).await.into_response()
    })
}
```

The key invariant: **`FromRequestParts` extractors share `&mut parts`** and must
run before `FromRequest`, because `FromRequest` may consume the body. Once the body
is consumed, `parts` cannot be reassembled into a new `Request`, so body extractors
must be the final argument.

### How `State<T>` extracts

**File:** `axum/src/extract/state.rs`

`State<T>` implements `FromRequestParts`. It reads `T` out of `Request::extensions`,
where it was inserted by `Router::with_state()` at build time via
`Endpoint::MethodRouter(method_router.with_state(state))`. This is zero-copy —
`T` is cloned from the `Arc`-backed extensions, not re-derived per request.

### How `Query<T>` extracts

**File:** `axum/src/extract/query.rs`

`Query<T>` also implements `FromRequestParts`. It calls
`parts.uri.query().unwrap_or_default()` and passes the raw query string to
`serde_html_form::from_str`, which deserialises it into `T`.

### How `Json<T>` extracts

**File:** `axum/src/json.rs:106`

`Json<T>` implements `FromRequest` (body-consuming):

```rust
async fn from_request(req: Request, state: &S) -> Result<Self, JsonRejection> {
    if !json_content_type(req.headers()) {
        return Err(MissingJsonContentType.into());   // 415
    }
    let bytes = Bytes::from_request(req, state).await?;   // buffer entire body
    Self::from_bytes(&bytes)                               // serde_json::from_slice
}
```

`json_content_type` (`json.rs:138`) validates that `Content-Type` is
`application/json` or any `application/*+json` suffix.

---

## Stage 6 — Rejection short-circuit

Every extractor's `Rejection` type implements `IntoResponse`. If any extractor
fails, its rejection is immediately converted to a `Response` and returned — the
handler function is never called, no further extractors run, and the response
travels back through middleware just like a successful response would.

Common rejections and their HTTP status codes:

| Extractor | Rejection | Status |
|---|---|---|
| `Json<T>` | `MissingJsonContentType` | 415 |
| `Json<T>` | `JsonSyntaxError` / `JsonDataError` | 422 |
| `Path<T>` | `FailedToDeserializePathParams` | 400 |
| `Path<T>` | `MissingPathParams` | 500 |
| `Query<T>` | `FailedToDeserializeQueryString` | 400 |
| `State<T>` | (infallible — panics at startup if state missing) | — |

---

## Stage 7 — Response conversion: `IntoResponse`

**File:** `axum-core/src/response/mod.rs`

The user's handler returns any type implementing `IntoResponse`. The trait has a
single method:

```rust
pub trait IntoResponse {
    fn into_response(self) -> Response;
}
```

`Response` here is `http::Response<axum::Body>`. The `impl_handler!` macro calls
`.into_response()` on the handler's return value as the final step, producing the
response that travels back up the call stack.

Implementations for common return types:

**`Json<T>`** (`axum/src/json.rs:201`): serialises `T` with `serde_json::to_writer`
into a `BytesMut` buffer (initial capacity 128 bytes), then sets
`Content-Type: application/json` and wraps the frozen bytes as the response body.

**`Html<T>`** (`axum/src/response/mod.rs`): sets `Content-Type: text/html` and
uses `T: Into<Full<Bytes>>` as the body.

**`Redirect`** (`axum/src/response/redirect.rs`): constructs a response with the
appropriate 3xx status and a `Location` header.

**`(StatusCode, T)`** tuples: axum implements `IntoResponse` for tuples of any
combination of `StatusCode`, `HeaderMap`, and body types, enabling concise handler
return values like `(StatusCode::CREATED, Json(user))`.

---

## Stage 8 — Response travels back through middleware

Once `Handler::call` resolves to a `Response`, the value propagates back up through
the Tower middleware stack in reverse order. Each `Layer` that processed the
request inbound now processes the response outbound — for example, `TraceLayer`
records the response status, `CompressionLayer` may compress the body, and
`CorsLayer` adds CORS headers. This is the standard Tower `Service` model: a
`Layer` wraps an inner service and can inspect or transform both the request (on
the way in) and the response (on the way out).

The final `Response` is handed back to `TowerToHyperService`
(`axum/src/serve/mod.rs:596`), which passes it to hyper for framing as HTTP/1.1 or
HTTP/2 bytes and writing to the `TokioIo` socket.

---

## Module interaction map

```
routing/mod.rs          Router<S>
  │  Service::call()    — entry point, wraps body, forwards to call_with_state
  │
routing/path_router.rs  PathRouter<S>
  │  call_with_state()  — matchit radix lookup, writes UrlParams to extensions
  │
routing/url_params.rs   insert_url_params()
  │                     — stores captured {params} in Request::extensions
  │
routing/method_routing.rs  MethodRouter<S>
  │  call_with_state()  — method check via call! macro, converts BoxedHandler → Route
  │
handler/mod.rs          Handler::call()  [impl_handler! macro]
  │                     — splits (parts, body), runs extractor loop
  │
axum-core/extract/      FromRequestParts / FromRequest traits
  ├── extract/state.rs  State<T>    reads from extensions (zero-copy)
  ├── extract/path/     Path<T>     reads UrlParams from extensions → serde
  ├── extract/query.rs  Query<T>    reads URI query string → serde
  └── json.rs           Json<T>     validates Content-Type, buffers body → serde_json
  │
  │  (rejection → IntoResponse short-circuit at any extractor)
  │
[user async fn]         handler(arg1, arg2, …) → impl IntoResponse
  │
axum-core/response/     IntoResponse::into_response()
  ├── json.rs           Json<T>     serde_json::to_writer → Bytes + Content-Type header
  ├── response/mod.rs   Html<T>     body + text/html header
  ├── response/redirect Redirect    3xx + Location header
  └── (StatusCode, …)   tuple impls
  │
Tower middleware stack  post-processing (tracing, compression, CORS headers)
  │
serve/mod.rs            TowerToHyperService → hyper → TCP bytes
```
