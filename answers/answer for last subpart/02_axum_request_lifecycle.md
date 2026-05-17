# Complete HTTP Request Lifecycle in Axum

## 1. Entry Point — `Router<()>` implements Tower `Service` (`mod.rs`)

```rust
// mod.rs
impl<B> Service<Request<B>> for Router<()> {
    fn poll_ready(&mut self, _: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        Poll::Ready(Ok(()))   // always ready — no backpressure at the router level
    }

    fn call(&mut self, req: Request<B>) -> Self::Future {
        let req = req.map(Body::new);   // erase the generic body type → axum::Body
        self.call_with_state(req, ())   // delegate with empty state
    }
}
```

`poll_ready` is a no-op; the router is stateless and always ready. `call` canonicalises the body to `axum::Body` (an `http_body::Body` eraser), then calls the internal method:

```rust
// mod.rs ~line 452
pub(crate) fn call_with_state(&self, req: Request, state: S) -> RouteFuture<Infallible> {
    let (req, state) = match self.inner.path_router.call_with_state(req, state) {
        Ok(future) => return future,          // path matched → done
        Err((req, state)) => (req, state),    // no path match → fall through
    };

    self.inner
        .catch_all_fallback
        .clone()
        .call_with_state(req, state)          // CONNECT / empty-path fallback
}
```

`RouterInner` holds two things: a `PathRouter<S>` and a `catch_all_fallback`. The catch-all handles the special `/{*__private__axum_fallback}` pattern registered for user-defined fallbacks, and the default 404 `NotFound` service.

---

## 2. Path Matching — `PathRouter::call_with_state()` (`path_router.rs`)

```rust
// path_router.rs ~line 325
pub(super) fn call_with_state(
    &self, mut req: Request, state: S,
) -> Result<RouteFuture<Infallible>, (Request, S)> {
    // ① Preserve the original URI before any middleware strips a prefix
    #[cfg(feature = "original-uri")]
    if req.extensions().get::<OriginalUri>().is_none() {
        req.extensions_mut().insert(OriginalUri(req.uri().clone()));
    }

    let (mut parts, body) = req.into_parts();

    match self.node.at(parts.uri.path()) {     // ② radix-tree lookup
        Ok(match_) => {
            let id = *match_.value;            // ③ RouteId (index into self.routes)

            #[cfg(feature = "matched-path")]
            set_matched_path_for_request(id, &self.node.route_id_to_path, &mut parts.extensions);

            url_params::insert_url_params(&mut parts.extensions, &match_.params); // ④

            let endpoint = self.routes.get(id.0).expect("bug");
            let req = Request::from_parts(parts, body);

            match endpoint {
                Endpoint::MethodRouter(mr) => Ok(mr.call_with_state(req, state)), // ⑤
                Endpoint::Route(route)     => Ok(route.clone().call_owned(req)),  // ⑥
            }
        }
        Err(MatchError::NotFound) => Err((Request::from_parts(parts, body), state)), // ⑦
    }
}
```

### Step-by-step

- **① `OriginalUri`** — inserted once into `Request::extensions` so that nested routers that later strip the prefix (via `StripPrefix` middleware) don't lose the original path for extractors.
- **② `self.node.at(path)`** — delegates to `matchit::Router::at`. `matchit` is a radix tree; it returns `Ok(Match { value: &RouteId, params: Params })` on success. This is an O(path-length) lookup.
- **③ `RouteId`** — a `usize` wrapper used as an index into `self.routes: Vec<Endpoint<S>>`. The bijective maps `route_id_to_path` and `path_to_route_id` inside `Node` allow the `matched-path` feature to recover the registered template (e.g. `/users/{id}`) from the numeric ID.
- **④ `url_params::insert_url_params`** — iterates `match_.params` (the captured segments like `id = "42"`) and stores them in `parts.extensions` under the key `UrlParams(Vec<(Arc<str>, PercentDecodedStr)>)`. This is how `Path<T>` extractors later recover `{id}`.
- **⑤ / ⑥** — route to a `MethodRouter` (most common) or a raw `Route` service.
- **⑦** — returns `Err((req, state))` if nothing matched; the router in `mod.rs` then tries the catch-all fallback.

---

## 3. How Extractors Read URL Parameters

After `call_with_state` inserts params into `extensions`, any handler parameter typed `Path<T>` uses:

```
axum/src/extract/path.rs → FromRequestParts::from_request_parts
  → req.extensions().get::<UrlParams>()
  → deserialise with serde
```

The `MatchedPath` extractor reads the `MatchedPath` extension set in step ③. `OriginalUri` reads the extension set in step ①. All extractor retrieval is zero-copy: the extensions map is a `TypeMap` on the already-allocated `Request`.

---

## 4. Method Dispatch — `MethodRouter::call_with_state()` (`method_routing.rs`)

```rust
// method_routing.rs ~line 1175
pub(crate) fn call_with_state(&self, req: Request, state: S) -> RouteFuture<E> {
    macro_rules! call {
        ($req:expr, $method_variant:ident, $svc:expr) => {
            if *req.method() == Method::$method_variant {
                match $svc {
                    MethodEndpoint::None => {}
                    MethodEndpoint::Route(route) => {
                        return route.clone().oneshot_inner_owned($req);
                    }
                    MethodEndpoint::BoxedHandler(handler) => {
                        let route = handler.clone().into_route(state); // bake state in
                        return route.oneshot_inner_owned($req);
                    }
                }
            }
        };
    }

    call!(req, HEAD, head);    // explicit HEAD handler first
    call!(req, HEAD, get);     // HEAD falls through to GET if no explicit HEAD
    call!(req, GET,  get);
    call!(req, POST, post);
    call!(req, OPTIONS, options);
    call!(req, PATCH, patch);
    call!(req, PUT, put);
    call!(req, DELETE, delete);
    call!(req, TRACE, trace);
    call!(req, CONNECT, connect);

    // nothing matched → 405 fallback
    let future = fallback.clone().call_with_state(req, state);
    match allow_header {
        AllowHeader::None  => future.allow_header(Bytes::new()),
        AllowHeader::Skip  => future,
        AllowHeader::Bytes(b) => future.allow_header(b.clone().freeze()),
    }
}
```

### Key details

- **`MethodEndpoint` enum** has three variants: `None` (method not registered), `Route` (already a fully-baked `tower::Service`), `BoxedHandler` (a handler function that still needs state injected). The `BoxedHandler` variant exists because handlers are registered before state is known; `into_route(state)` finalises them.
- **HEAD auto-fallback** — `call!(req, HEAD, get)` is checked second, so a `GET` handler transparently services `HEAD`. The response body is stripped later in `RouteFuture`.
- **`Allow` header** — during registration, `on_endpoint` calls `append_allow_header` for every method being registered. At call time the accumulated `AllowHeader::Bytes` value is attached to the 405 fallback future so the response automatically carries the correct `Allow: GET, POST, ...` header.
- The default fallback is `StatusCode::METHOD_NOT_ALLOWED` wrapped in a `service_fn`.

---

## 5. Handler Execution and Response Conversion

Once a `Route` is selected, `.oneshot_inner_owned(req)` drives the Tower service to completion, returning a `RouteFuture<E>`. The future resolves to `Response` (which is `http::Response<axum::Body>`).

For handler functions (`async fn handler(...) -> impl IntoResponse`):

```
Handler::call(req, state)
  → extract all parameters via FromRequest / FromRequestParts
  → call the async fn
  → return value.into_response() → http::Response<Body>
```

`IntoResponse` is implemented for all common return types (`StatusCode`, `(StatusCode, impl IntoResponse)`, `Json<T>`, `Html<T>`, `String`, `&'static str`, `Result<T, E>`, …). Each converts itself into `http::Response<Body>` by setting appropriate status codes and `Content-Type` headers.

The final `http::Response<Body>` travels back through any middleware layers (each layer wraps `Route` at registration time via `Layer::layer`), then back through Tower's `Service::call` chain to the TCP connection handler in `axum::serve`.

---

## Data-Flow Summary

```
TCP bytes
  └─ axum::serve → hyper → http::Request<B>
       └─ Router::call()                             [mod.rs]
            └─ Router::call_with_state()             [mod.rs]
                 └─ PathRouter::call_with_state()    [path_router.rs]
                      ├─ OriginalUri → extensions
                      ├─ matchit radix-tree lookup
                      ├─ MatchedPath → extensions    (feature flag)
                      ├─ url_params → extensions     (Path extractor source)
                      └─ MethodRouter::call_with_state()   [method_routing.rs]
                           ├─ method switch (HEAD→GET fallback)
                           ├─ BoxedHandler::into_route(state)
                           └─ Route::oneshot_inner_owned()
                                └─ Handler::call → extract → async fn → IntoResponse
                                     └─ http::Response<Body>
  ← TCP bytes
```

---

## Source Files Referenced

| File | Role |
|------|------|
| `axum/src/routing/mod.rs` | `Router` struct, Tower `Service` impl, `call_with_state`, fallback wiring |
| `axum/src/routing/path_router.rs` | `PathRouter`, `matchit` radix-tree wrapper, `OriginalUri`/`UrlParams`/`MatchedPath` insertion |
| `axum/src/routing/method_routing.rs` | `MethodRouter`, per-method dispatch, `MethodEndpoint`, `Allow` header accumulation |
