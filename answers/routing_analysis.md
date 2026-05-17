# Prompt
> Analyze the current routing implementation in axum. Focus only on the modules related to Router, MethodRouter, and route matching. Do not rewrite code. Identify the main abstractions, explain their responsibilities, and suggest at most two small refactoring opportunities with file paths and line references.

---

# Routing Implementation Analysis

## Main abstractions and their responsibilities

### `Router<S>` — `routing/mod.rs:86`
The public-facing builder type. Wraps `Arc<RouterInner<S>>` so cloning is cheap
(one atomic increment). The type parameter `S` represents *missing* state — only
`Router<()>` can be passed to `serve()`, enforced at compile time via the `Service`
impl at `mod.rs:599`. Every builder method (`route`, `nest`, `merge`, `layer`,
`fallback`) peels off the `Arc` via `into_inner()` (`mod.rs:172`), mutates, and
re-wraps — the two macros `map_inner!` and `tap_inner!` (`mod.rs:129–152`) exist
solely to encapsulate that pattern.

### `RouterInner<S>` — `routing/mod.rs:98`
The actual data: `path_router`, a `default_fallback` boolean flag (distinguishes
the default 404 from a user-supplied fallback), and `catch_all_fallback`. The flag
exists so `merge` knows whether it is safe to override a fallback from one side.

### `PathRouter<S>` — `routing/path_router.rs:16`
Owns route storage and the path tree. Two structures live side by side:
- `routes: Vec<Endpoint<S>>` — the endpoints, indexed by `RouteId(usize)`
- `node: Arc<Node>` — the matchit radix tree plus two reverse-lookup `HashMap`s

The `Vec` index and the `RouteId` integer are the same value; `new_route` assigns
`RouteId(self.routes.len())` before pushing.

### `Node` — `routing/path_router.rs:406`
A thin wrapper around `matchit::Router<RouteId>` that maintains two `HashMap`s:
- `route_id_to_path` — used in `merge` and `nest` to iterate routes of another
  `PathRouter` by walking their `RouteId`s and finding the path strings to re-insert
- `path_to_route_id` — used in `route()` (`path_router.rs:73`) to detect whether
  a path already exists, so two `MethodRouter`s for the same path are merged rather
  than duplicated

`matchit` itself has no API for "does this path exist?", so the second `HashMap`
fills that gap.

### `MethodRouter<S, E>` — `routing/method_routing.rs`
Holds one `MethodEndpoint<S, E>` per HTTP verb (9 fields: GET, HEAD, POST, PUT,
DELETE, PATCH, OPTIONS, TRACE, CONNECT), a `fallback`, and an `allow_header`.
Dispatch in `call_with_state` (`method_routing.rs:1167`) is done by the `call!`
macro expanding to a sequence of `if req.method() == Method::X` checks — the
destructuring at line 1190 is a deliberate pattern that ensures no field can be
silently omitted when a new method is added.

### `Fallback<S, E>` — `routing/mod.rs:710`
Three variants: `Default` (the built-in 404), `Service` (a user-supplied Tower
service), and `BoxedHandler` (a user-supplied handler not yet bound to state).
`merge` returns `None` if both sides are non-`Default`, which is the signal to
panic.

### `Endpoint<S>` (referenced in `path_router.rs`)
The union type stored in `PathRouter::routes`. Either a `MethodRouter` (the common
case) or a raw `Route` (for `route_service`).

---

## Two small refactoring opportunities

### 1. Extract the duplicated closure in `fallback_endpoint`

**File:** `axum/src/routing/mod.rs`
**Lines:** 398–437

`fallback_endpoint` registers the fallback at two paths (`"/"` and
`FALLBACK_PARAM_PATH`). Both calls wrap the endpoint in an **identical** `layer_fn`
closure:

```rust
// first registration — mod.rs:401–416
endpoint.clone().layer(
    layer_fn(|service: Route| {
        let mut service = Some(service);
        service_fn(move |mut request: Request| {
            #[cfg(feature = "matched-path")]
            request.extensions_mut().remove::<MatchedPath>();
            let route = take_route_or_internal_error(&mut service);
            route.oneshot_inner_owned(request)
        })
    })
)

// second registration — mod.rs:421–436  (same closure, copy-pasted)
endpoint.layer(
    layer_fn(|service: Route| {
        let mut service = Some(service);
        service_fn(move |mut request: Request| {
            #[cfg(feature = "matched-path")]
            request.extensions_mut().remove::<MatchedPath>();
            let route = take_route_or_internal_error(&mut service);
            route.oneshot_inner_owned(request)
        })
    })
)
```

The code acknowledges this with a `// TODO make this better.` comment at
`mod.rs:392`. The closure could be extracted into a named
`fn make_fallback_layer() -> impl Layer<Route>` helper, eliminating the duplication
and resolving the TODO.

---

### 2. Replace the 9-field empty check in `MethodRouter::route_layer` with a `has_routes()` method

**File:** `axum/src/routing/method_routing.rs`
**Lines:** 1053–1066

```rust
if self.get.is_none()
    && self.head.is_none()
    && self.delete.is_none()
    && self.options.is_none()
    && self.patch.is_none()
    && self.post.is_none()
    && self.put.is_none()
    && self.trace.is_none()
    && self.connect.is_none()
{
    panic!("Adding a route_layer before any routes is a no-op. …");
}
```

This is a manual 9-field conjunction. If a new HTTP method is ever added to
`MethodRouter`, this check must be updated by hand — there is no compiler
enforcement. Compare `PathRouter`, which already has an equivalent extracted as
`has_routes()` at `path_router.rs:301–303`:

```rust
pub(super) fn has_routes(&self) -> bool {
    !self.routes.is_empty()
}
```

Adding an equivalent `has_routes()` method to `MethodRouter` would centralise the
logic and mirror the existing pattern. The `call_with_state` destructuring at
`method_routing.rs:1190` already shows the team prefers exhaustive patterns over
ad-hoc field lists for exactly this reason.
