# axum — End-to-End Request Trace

## Endpoint under trace

```rust
// POST /users  →  create_user(State<AppState>, Json<CreateUser>) -> Json<User>
Router::new()
    .route("/users", post(create_user))
    .with_state(app_state)
```

This is the canonical axum pattern: a `POST` handler that reads shared state,
deserialises a JSON body, and returns a JSON response.  
Every step below maps to the actual source files and line numbers in the axum repo.

---

## Step 1 — Server startup: `axum::serve()`

**File:** `axum/src/serve/mod.rs`

```
L103  pub fn serve<L, M, S, B>(listener, make_service) -> Serve<…>
```

`serve()` returns a `Serve` struct (not yet running). Calling `.await` on it
(via `IntoFuture`) drives:

```
L321  async fn Serve::run(self) -> !
L333    let (io, remote_addr) = listener.accept().await   // blocks until TCP SYN
L334    handle_connection(&mut make_service, …, io, remote_addr, &executor)
```

The accept loop is infinite (`-> !`); each accepted socket is dispatched to:

```
L559  async fn handle_connection(make_service, …, io, remote_addr, executor)
L580    trace!("connection {remote_addr:?} accepted")
L587    let tower_service = make_service.call(IncomingStream { io, remote_addr })
          // clones the Router for this connection
L596    let hyper_service = TowerToHyperService::new(tower_service)
          // wraps Tower Service → hyper-compatible Service
L601    executor.execute(async move {
L603      Builder::new(hyper_executor)          // hyper_util auto-builder (HTTP/1 + HTTP/2)
L604        .http1().timer(TokioTimer::new())   // per-connection request header timeout
          …
          .serve_connection_with_upgrades(io, hyper_service)
        })
```

`executor.execute` spawns the connection task via `tokio::spawn`.

---

## Step 2 — Hyper parses bytes → calls Tower Service

**External:** `hyper_util::service::TowerToHyperService`

Hyper reads raw bytes from the socket, assembles an `http::Request<Incoming>`,
then calls `TowerToHyperService::call(request)`, which delegates to the wrapped
Tower Service (our `Router`).

Back in `handle_connection` the body stream is re-wrapped:

```
L594  .map_request(|req: Request<Incoming>| req.map(Body::new))
```

`Body::new` (`axum/src/body/mod.rs`) wraps hyper's streaming `Incoming` body
into axum's `Body` newtype.

---

## Step 3 — `Router` receives the request

**File:** `axum/src/routing/mod.rs`

```
L609  impl<B> Service<Request<B>> for Router<()>
L614    fn call(&mut self, req: Request<B>) -> Self::Future {
L615      let req = req.map(Body::new)
L616      self.call_with_state(req, ())     // state already baked in
        }
```

`call_with_state` immediately forwards to `PathRouter`:

```
L452  pub(crate) fn call_with_state(&self, req, state) -> RouteFuture<Infallible>
L453    match self.inner.path_router.call_with_state(req, state) {
L454      Ok(future) => return future,       // path matched
L456      Err((req, state)) => …             // fall through to catch_all_fallback (404)
        }
```

---

## Step 4 — Path matching via `PathRouter` + `matchit`

**File:** `axum/src/routing/path_router.rs`

```
L325  pub(super) fn call_with_state(&self, mut req, state)
        -> Result<RouteFuture<Infallible>, (Request, S)>

L342    match self.node.at(parts.uri.path()) {   // matchit radix-tree lookup
          Ok(match_) => {
            let id = *match_.value               // RouteId

L346        // (feature = "matched-path")
            set_matched_path_for_request(id, …) // inserts "/users" into extensions

L353        url_params::insert_url_params(       // inserts captured {id}, {*rest}, etc.
              &mut parts.extensions, &match_.params)

L358        let endpoint = self.routes.get(id.0) // look up MethodRouter by RouteId

L362        Endpoint::MethodRouter(method_router) =>
              Ok(method_router.call_with_state(req, state))
          }
          Err(MatchError::NotFound) => Err((req, state))  // caller uses fallback
        }
```

For `POST /users` the tree lookup succeeds immediately; captured URL params
are inserted into `Request::extensions` so extractors can read them later.

---

## Step 5 — Method dispatch via `MethodRouter`

**File:** `axum/src/routing/method_routing.rs`

```
L1167  pub(crate) fn call_with_state(&self, req, state) -> RouteFuture<E>
         // macro call! checks req.method() == Method::POST
L1193    call!(req, POST, post);             // matches our handler
           // MethodEndpoint::BoxedHandler(handler) branch:
           let route = handler.clone().into_route(state);
           return route.oneshot_inner_owned(req);
```

The `call!` macro expands inline for each HTTP method. The matched
`BoxedHandler` is converted to a `Route` (Tower Service) via `into_route(state)`,
then driven with `.oneshot_inner_owned(req)`.

---

## Step 6 — `Handler::call` — extractor loop

**File:** `axum/src/handler/mod.rs`

The `impl_handler!` macro generates a `Handler` impl for every arity
(0 to 16 arguments). For our handler `(State<AppState>, Json<CreateUser>)`:

```
L227  impl<F, Fut, S, Res, M, $($ty,)* $last> Handler<(M, $($ty,)* $last,), S> for F
        // $ty  = State<AppState>   (implements FromRequestParts)
        // $last = Json<CreateUser> (implements FromRequest — consumes body)

L238  fn call(self, req: Request, state: S) -> Self::Future {
L239    let (mut parts, body) = req.into_parts()   // split head from body
        Box::pin(async move {

          // ── FromRequestParts loop (non-body extractors) ──────────────
L242      let State(app_state) =
            match State::from_request_parts(&mut parts, &state).await {
              Ok(value) => value,
              Err(rejection) => return rejection.into_response(),  // short-circuit
            };

          // ── FromRequest (body extractor, always last) ────────────────
L248      let req = Request::from_parts(parts, body)  // reassemble

L250      let Json(payload) =
            match Json::from_request(req, &state).await {
              Ok(value) => value,
              Err(rejection) => return rejection.into_response(),
            };

          // ── invoke user fn ───────────────────────────────────────────
          self(State(app_state), Json(payload)).await.into_response()
        })
      }
```

Key invariant: `FromRequestParts` extractors run first on the shared
`parts`; the single `FromRequest` extractor (the one that may consume the body)
runs last.

---

## Step 7 — `State<T>` extraction

**File:** `axum/src/extract/state.rs`

`State<T>` implements `FromRequestParts`. It reads `T` out of
`Request::extensions`, which was inserted when the Router called
`.with_state(app_state)`. No body access required.

---

## Step 8 — `Json<T>` extraction

**File:** `axum/src/json.rs`

```
L99   impl<T, S> FromRequest<S> for Json<T>
L106    async fn from_request(req, state) -> Result<Self, JsonRejection>
L107      if !json_content_type(req.headers()) {
            return Err(MissingJsonContentType.into());  // 415 Unsupported Media Type
          }

          // buffer the entire body into Bytes
L111      let bytes = Bytes::from_request(req, state).await?;
          Self::from_bytes(&bytes)
```

`from_bytes` (L163):

```
L184      let mut deserializer = serde_json::Deserializer::from_slice(&bytes);
          serde_path_to_error::deserialize(&mut deserializer)
            // wraps serde_json errors with the field path that failed
            .map_err(make_rejection)   // → JsonDataError or JsonSyntaxError
          // + checks no trailing bytes remain
```

`json_content_type` (L138): validates `Content-Type` starts with
`application/json` or ends with `+json` (e.g. `application/cloudevents+json`).

**Rejection path** (any of the above fail):
- `MissingJsonContentType` → `415 Unsupported Media Type`
- `JsonSyntaxError` / `JsonDataError` → `422 Unprocessable Entity`
- Body buffering error → `400 Bad Request` / `413 Payload Too Large`

---

## Step 9 — User handler executes

```rust
async fn create_user(
    State(db): State<AppState>,
    Json(payload): Json<CreateUser>,
) -> Json<User> {
    let user = db.create(payload).await;
    Json(user)
}
```

This is pure application code. axum does not touch it except to call it and
collect its return value.

---

## Step 10 — `Json<T>` serialises the response

**File:** `axum/src/json.rs`

```
L197  impl<T: Serialize> IntoResponse for Json<T>
L201    fn into_response(self) -> Response
L228      let mut buf = BytesMut::with_capacity(128).writer()
L229      let res = serde_json::to_writer(&mut buf, &self.0)
          make_response(buf.into_inner(), res)
```

`make_response` (L203):
- **Success** → `200 OK` + `Content-Type: application/json` + serialised bytes
- **Serialize error** → `500 Internal Server Error` + plain-text error body

---

## Step 11 — Response travels back

The `Response` returned from `Handler::call` propagates back through:

1. `MethodRouter::call_with_state` → wraps in `RouteFuture`
2. Tower middleware layers (if any) post-process headers/body
3. `TowerToHyperService` hands the response back to hyper
4. Hyper frames it as HTTP bytes and writes to the `TokioIo` socket

---

## Full call-stack summary

```
tokio::net::TcpListener::accept()              (OS / Tokio)
  axum/src/serve/mod.rs:333   Serve::run()     – accept loop
  axum/src/serve/mod.rs:559   handle_connection()
  axum/src/serve/mod.rs:596   TowerToHyperService::new(router)
    hyper_util  –  HTTP framing, TLS upgrades, keep-alive
  axum/src/routing/mod.rs:614       Router::call()
  axum/src/routing/mod.rs:452       Router::call_with_state()
  axum/src/routing/path_router.rs:325  PathRouter::call_with_state()
  axum/src/routing/path_router.rs:342    matchit node.at("/users")  → RouteId
  axum/src/routing/path_router.rs:353    url_params::insert_url_params()
  axum/src/routing/method_routing.rs:1167  MethodRouter::call_with_state()
  axum/src/routing/method_routing.rs:1193    call!(req, POST, post)
  axum/src/handler/mod.rs:238     Handler::call()   [macro-generated]
  axum/src/handler/mod.rs:242       State::from_request_parts()   → AppState
  axum/src/handler/mod.rs:250       Json::from_request()
  axum/src/json.rs:107                json_content_type() check
  axum/src/json.rs:111                Bytes::from_request()  – buffer body
  axum/src/json.rs:184                serde_json::Deserializer::from_slice()
                                      user async fn create_user(…)
  axum/src/json.rs:201              Json::into_response()
  axum/src/json.rs:229                serde_json::to_writer()
  axum/src/json.rs:203                → 200 OK + Content-Type: application/json
    hyper_util  –  serialise response bytes → socket write
```
