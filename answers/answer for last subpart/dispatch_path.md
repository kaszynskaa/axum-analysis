# Prompt
> Focus only on the request dispatch path inside:
> - axum/src/routing/mod.rs
> - axum/src/routing/path_router.rs
> - axum/src/routing/method_routing.rs
>
> Explain how Router dispatches requests and selects handlers. Do not discuss
> extractors, middleware, or response conversion unless directly required.

---

# Request Dispatch Path

## The three types and how they chain

```
Router<S>          mod.rs          — public Tower Service; owns state + fallback
  └── PathRouter<S>  path_router.rs  — owns radix tree + endpoint vec
        └── MethodRouter<S>  method_routing.rs  — owns one handler per HTTP method
```

Each type is responsible for exactly one decision: `Router` decides whether the
request is routable at all; `PathRouter` decides which endpoint owns the path;
`MethodRouter` decides which handler owns the method.

---

## 1. `Router::call()` — entry point

**`axum/src/routing/mod.rs:614`**

```rust
fn call(&mut self, req: Request<B>) -> Self::Future {
    let req = req.map(Body::new);
    self.call_with_state(req, ())
}
```

`Router` implements `Service<Request<B>>` only for `Router<()>` — i.e. only after
state has been sealed with `.with_state()`. The body is re-wrapped from hyper's
`Incoming` into axum's `Body` newtype, then forwarded immediately to
`call_with_state`.

**`mod.rs:452`**

```rust
pub(crate) fn call_with_state(&self, req: Request, state: S) -> RouteFuture<Infallible> {
    match self.inner.path_router.call_with_state(req, state) {
        Ok(future)       => return future,
        Err((req, state)) => (req, state),
    };
    self.inner.catch_all_fallback.clone().call_with_state(req, state)
}
```

`Router` does no path inspection itself. It delegates the entire decision to
`PathRouter` and only acts if `PathRouter` returns `Err` — meaning no path matched
— in which case it invokes the catch-all fallback (default 404, or a handler
registered via `.fallback()`).

---

## 2. `PathRouter::call_with_state()` — path matching

**`axum/src/routing/path_router.rs:325`**

`PathRouter` owns two data structures populated at startup:

```rust
struct PathRouter<S> {
    routes: Vec<Endpoint<S>>,   // indexed by RouteId
    node:   Arc<Node>,          // matchit radix tree + two reverse HashMaps
    v7_checks: bool,
}
```

`Node` (`path_router.rs:406`) wraps `matchit::Router<RouteId>`. Every
`.route(path, handler)` call inserts the path string into the radix tree and
pushes the endpoint into `routes`, assigning `RouteId(routes.len())` as the key.
The two `HashMap`s in `Node` (`route_id_to_path` and `path_to_route_id`) are used
only during startup for merge/nest operations — not during request dispatch.

At request time the dispatch logic is:

```rust
// path_router.rs:340–371
let (mut parts, body) = req.into_parts();

match self.node.at(parts.uri.path()) {
    Ok(match_) => {
        let id = *match_.value;   // RouteId from the tree

        // (matched-path feature) store "/users/{id}" string in extensions
        set_matched_path_for_request(id, &self.node.route_id_to_path, &mut parts.extensions);

        // store captured {param} values in extensions
        url_params::insert_url_params(&mut parts.extensions, &match_.params);

        let endpoint = self.routes.get(id.0)
            .expect("no route for id. This is a bug in axum.");

        let req = Request::from_parts(parts, body);
        match endpoint {
            Endpoint::MethodRouter(mr) => Ok(mr.call_with_state(req, state)),
            Endpoint::Route(route)     => Ok(route.clone().call_owned(req)),
        }
    }
    Err(MatchError::NotFound) => Err((Request::from_parts(parts, body), state)),
}
```

### Why the request is split before the lookup

The request is split into `(parts, body)` at line 340 so that
`url_params::insert_url_params` and `set_matched_path_for_request` can write into
`parts.extensions` without moving or cloning the body. It is reassembled with
`Request::from_parts` immediately before being forwarded.

### The two endpoint types

`Endpoint<S>` is an enum with two variants:

- **`Endpoint::MethodRouter`** — the common case, registered via `.route(path, get(f).post(g))`. Forwarded to `MethodRouter::call_with_state` for method-level dispatch.
- **`Endpoint::Route`** — a raw Tower `Service`, registered via `.route_service(path, svc)`. Bypasses `MethodRouter` entirely and is called directly with `.call_owned(req)`.

### No match → `Err`

If the radix tree returns `MatchError::NotFound`, `PathRouter` returns `Err((req, state))` to `Router::call_with_state`, which then invokes `catch_all_fallback`. The fallback itself is not a concern of `PathRouter`.

---

## 3. `MethodRouter::call_with_state()` — method dispatch

**`axum/src/routing/method_routing.rs:1167`**

`MethodRouter` stores nine `MethodEndpoint<S, E>` fields — one per HTTP method —
plus a `fallback` and an `allow_header`:

```rust
struct MethodRouter<S, E = Infallible> {
    get:     MethodEndpoint<S, E>,
    head:    MethodEndpoint<S, E>,
    post:    MethodEndpoint<S, E>,
    put:     MethodEndpoint<S, E>,
    delete:  MethodEndpoint<S, E>,
    patch:   MethodEndpoint<S, E>,
    options: MethodEndpoint<S, E>,
    trace:   MethodEndpoint<S, E>,
    connect: MethodEndpoint<S, E>,
    fallback:     Fallback<S, E>,
    allow_header: AllowHeader,
}
```

Each `MethodEndpoint` is one of three variants:
- `None` — method not registered
- `Route(Route)` — a pre-built Tower `Service` (from `.get_service(svc)`)
- `BoxedHandler(BoxedIntoRoute)` — a handler not yet bound to state (from `.get(handler)`)

Dispatch is a cascade of `if req.method() == …` checks via the `call!` macro:

```rust
// method_routing.rs:1168–1213
macro_rules! call {
    ($req:expr, $method_variant:ident, $svc:expr) => {
        if *req.method() == Method::$method_variant {
            match $svc {
                MethodEndpoint::None => {}   // not registered, fall through
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
call!(req, HEAD, get);     // HEAD falls back to GET if no HEAD handler registered
call!(req, GET,  get);
call!(req, POST, post);
call!(req, OPTIONS, options);
call!(req, PATCH, patch);
call!(req, PUT, put);
call!(req, DELETE, delete);
call!(req, TRACE, trace);
call!(req, CONNECT, connect);
```

Three things are notable here:

1. **HEAD → GET fallback.** `call!(req, HEAD, head)` is checked first. If no
   explicit HEAD handler is registered (`MethodEndpoint::None`), the macro falls
   through and `call!(req, HEAD, get)` tries the GET handler for HEAD requests.
   This is how axum automatically serves HEAD from GET without any user
   configuration.

2. **`BoxedHandler::into_route(state)`** binds the cloned state to the handler at
   dispatch time, producing a `Route`. State is stored in the `BoxedIntoRoute`
   until this point — it is not baked in at startup unless `.with_state()` was
   called explicitly, which converts all `BoxedHandler` variants to `Route` eagerly
   (`method_routing.rs:820`).

3. **Destructuring for exhaustiveness.** Before the macro calls, `self` is
   destructured by name (`mod.rs:1190`):
   ```rust
   let Self { get, head, delete, options, patch, post, put, trace, connect, fallback, allow_header } = self;
   ```
   This is intentional: if a new field is ever added to `MethodRouter`, the
   compiler will force an update here. The `call!` cascade cannot silently omit
   a new method.

### No method match → fallback + `Allow` header

If all nine `call!` checks fall through, the request is handed to `fallback`:

```rust
// method_routing.rs:1215–1221
let future = fallback.clone().call_with_state(req, state);

match allow_header {
    AllowHeader::None    => future.allow_header(Bytes::new()),
    AllowHeader::Skip    => future,
    AllowHeader::Bytes(b) => future.allow_header(b.clone().freeze()),
}
```

The default fallback returns `405 Method Not Allowed`. The `allow_header` is a
`BytesMut` built at startup by `append_allow_header` (`method_routing.rs:1225`)
every time a method is registered — it accumulates a comma-separated list of
allowed methods (e.g. `GET, HEAD, POST`) and is frozen into `Bytes` before being
set on the response.

---

## Complete dispatch call chain

```
Router::call()                          mod.rs:614
  req.map(Body::new)                    — wrap hyper body
  Router::call_with_state()             mod.rs:452
    PathRouter::call_with_state()       path_router.rs:325
      req.into_parts()                  — split to allow extension writes
      matchit node.at(uri.path())       path_router.rs:342  — radix lookup
      insert_url_params()               path_router.rs:353  — write captures to extensions
      routes.get(id.0)                  path_router.rs:355  — O(1) vec index
      MethodRouter::call_with_state()   method_routing.rs:1167
        call!(req, HEAD, head)          — method check 1
        call!(req, HEAD, get)           — method check 2 (HEAD→GET fallback)
        call!(req, GET,  get)           — method check 3
        …                              — checks 4–9
        BoxedHandler::into_route(state) — bind state to handler → Route
        route.oneshot_inner_owned(req)  — drive handler to completion
      [or if no method match]
        fallback.call_with_state()      — 405 + Allow header
    [or if no path match]
      catch_all_fallback.call_with_state()  mod.rs:458  — 404 or custom fallback
```

---

## Key design decisions

**`RouteId` is the `Vec` index.** `PathRouter` uses `RouteId(usize)` as both
the value stored in the radix tree and the index into `routes`. The tree lookup
returns a `RouteId`; the endpoint is fetched with `routes.get(id.0)`. This
makes the per-request path from tree match to handler retrieval O(1) with no
`HashMap` involved.

**State is deferred.** `BoxedHandler` holds the handler function without state.
State is bound at the last possible moment — `into_route(state)` inside
`call_with_state` — unless `.with_state()` was called at build time, which
converts all `BoxedHandler` variants to `Route` eagerly and avoids the per-request
clone of the handler.

**Fallback separation.** `Router` has a `catch_all_fallback` for unmatched paths;
`MethodRouter` has its own `fallback` for unmatched methods. They are independent
and can be configured separately via `Router::fallback()` and
`MethodRouter::fallback_service()`.
