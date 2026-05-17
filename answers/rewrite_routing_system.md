# Rewrite the Routing System

## Assessment

"Rewrite the routing system" is a **high-risk, high-complexity task** that touches
the core of axum. Before writing a single line, the scope must be narrowed and the
motivation made explicit — otherwise the rewrite will break the entire framework.

---

## Current routing system (what exists today)

### Files involved

| File | Responsibility |
|---|---|
| `axum/src/routing/mod.rs` | `Router<S>` — public API, composition, `.with_state()` |
| `axum/src/routing/path_router.rs` | `PathRouter` — owns the matchit tree, maps path → endpoint |
| `axum/src/routing/method_routing.rs` | `MethodRouter` — dispatches on HTTP method |
| `axum/src/routing/route.rs` | `Route` — type-erased Tower `Service` for one endpoint |
| `axum/src/routing/method_filter.rs` | `MethodFilter` — bitmask of allowed HTTP methods |
| `axum/src/routing/url_params.rs` | Stores/reads captured path params in `Request::extensions` |
| `axum/src/routing/strip_prefix.rs` | Strips path prefix for nested routers |
| `axum/src/routing/into_make_service.rs` | `IntoMakeService` — clones Router per connection |
| `axum/src/routing/future.rs` | `RouteFuture` — type-erased response future |
| `axum/src/routing/not_found.rs` | Default 404 fallback handler |

### How it works (summary)

```
Router<S>
  └── PathRouter<S>
        ├── matchit::Router<RouteId>   ← radix tree, path → RouteId
        └── Vec<Endpoint<S>>           ← RouteId index → MethodRouter or Route

MethodRouter<S>
  ├── MethodEndpoint for GET, POST, PUT, DELETE, PATCH, …
  └── fallback: Fallback<S>
```

1. `PathRouter` inserts paths into a `matchit` radix tree at startup.
2. On each request, `matchit::Router::at(path)` does an O(n) radix lookup.
3. Captured params are stored in `Request::extensions` via `url_params`.
4. The matched `MethodRouter` checks `req.method()` via the `call!` macro.
5. The matched `BoxedHandler` is converted to a `Route` and driven with `.oneshot()`.

---

## Clarifying the rewrite goal

Before rewriting, answer these questions:

| Question | Impact |
|---|---|
| Replace `matchit` with a different algorithm? | Only `path_router.rs` changes |
| Change how method dispatch works? | Only `method_routing.rs` changes |
| Change the `Router<S>` public API? | Breaking change for all axum users |
| Change how state is passed? | Touches `Router`, extractors, handler macro |
| Change `Route` / `RouteFuture`? | Internal only, lower risk |

---

## Proposed rewrite plan (path matching replacement as example)

The most self-contained rewrite: **replace `matchit` with a custom trie** while
keeping the public API identical.

### Step 1 — Define the new matcher interface

Create `axum/src/routing/matcher.rs`:

```rust
pub(crate) trait RouteMatcher: Send + Sync + 'static {
    fn insert(&mut self, path: &str, id: RouteId) -> Result<(), String>;
    fn at<'p>(&self, path: &'p str) -> Option<Match<'p>>;
}

pub(crate) struct Match<'p> {
    pub route_id: RouteId,
    pub params: Vec<(&'p str, &'p str)>,  // (name, value) pairs
}
```

This decouples `PathRouter` from `matchit` and allows swapping implementations.

### Step 2 — Wrap the existing matchit in the new trait

```rust
// axum/src/routing/matcher/matchit_impl.rs
pub(crate) struct MatchitMatcher {
    inner: matchit::Router<RouteId>,
    param_names: HashMap<RouteId, Vec<String>>,
}

impl RouteMatcher for MatchitMatcher { … }
```

All existing tests pass — no behaviour change yet.

### Step 3 — Implement the replacement

```rust
// axum/src/routing/matcher/trie.rs
pub(crate) struct TrieMatcher { … }
impl RouteMatcher for TrieMatcher { … }
```

### Step 4 — Swap in `PathRouter`

```rust
// axum/src/routing/path_router.rs
pub(super) struct PathRouter<S> {
    routes: Vec<Endpoint<S>>,
-   node: Arc<Node>,          // matchit-specific
+   matcher: Arc<dyn RouteMatcher>,
    v7_checks: bool,
}
```

### Step 5 — Update `call_with_state`

```rust
// path_router.rs:325
match self.matcher.at(parts.uri.path()) {
    Some(m) => {
        url_params::insert_url_params(&mut parts.extensions, &m.params);
        …
    }
    None => Err((Request::from_parts(parts, body), state)),
}
```

### Step 6 — Run the test suite

```bash
cargo test --workspace
```

All 635 existing tests must pass before the rewrite is considered complete.

---

## Files changed summary

| File | Change type |
|---|---|
| `axum/src/routing/matcher.rs` | **New** — trait definition |
| `axum/src/routing/matcher/matchit_impl.rs` | **New** — existing behaviour wrapped |
| `axum/src/routing/matcher/trie.rs` | **New** — replacement implementation |
| `axum/src/routing/path_router.rs` | **Modified** — use `dyn RouteMatcher` |
| `axum/src/routing/mod.rs` | **Modified** — expose new matcher in `Router` builder (optional) |
| All other files | **Unchanged** |

---

## What NOT to rewrite (without separate RFC)

| Component | Reason to leave alone |
|---|---|
| `Router<S>` public API | Breaking change for every axum application |
| `MethodRouter` method dispatch | Stable, well-tested, no known issues |
| `url_params` extension storage | Shared contract between `PathRouter` and `Path<T>` extractor |
| `Route` / `RouteFuture` | Internal types but tightly coupled to Tower `Service` |
| State management via `extensions` | Changing this breaks `State<T>` extractor |

---

## Risk assessment

| Risk | Mitigation |
|---|---|
| Regression in path matching | 635 existing tests + add property-based tests with `proptest` |
| Performance regression | Benchmark with `criterion` before and after |
| Breaking nested route / prefix stripping | `routing/tests/nest.rs` covers these cases |
| Breaking wildcard captures | `routing/tests/mod.rs` has wildcard tests |
