# Prompt
> Rewrite the routing system.

---

# Rewrite the Routing System — Improved Answer

## Why this prompt needs scoping first

"Rewrite the routing system" is dangerously broad. The axum routing system is the
**core of the framework** — a full rewrite without scoping would:
- Break the public `Router<S>` API used by every axum application
- Invalidate all 635 existing tests
- Risk subtle regressions in path matching, method dispatch, and state propagation

The right first step is always: **what specifically is broken or insufficient, and why?**

---

## Current routing system — complete map

### File inventory

| File | Lines | Responsibility |
|---|---|---|
| `axum/src/routing/mod.rs` | ~750 | `Router<S>` public API; `.route()`, `.nest()`, `.merge()`, `.layer()`, `.with_state()` |
| `axum/src/routing/path_router.rs` | ~430 | `PathRouter` — owns the `matchit` radix tree; maps path → `RouteId` → `Endpoint` |
| `axum/src/routing/method_routing.rs` | ~1700 | `MethodRouter` — per-method dispatch via `call!` macro; builds `get()`, `post()`, … |
| `axum/src/routing/route.rs` | ~150 | `Route` — type-erased Tower `Service` for one endpoint |
| `axum/src/routing/method_filter.rs` | ~100 | `MethodFilter` — bitmask of allowed HTTP methods |
| `axum/src/routing/url_params.rs` | ~60 | Stores/reads captured path params in `Request::extensions` |
| `axum/src/routing/strip_prefix.rs` | ~80 | Strips path prefix for nested routers |
| `axum/src/routing/into_make_service.rs` | ~60 | `IntoMakeService` — clones `Router` per connection |
| `axum/src/routing/future.rs` | ~100 | `RouteFuture` — type-erased response future |
| `axum/src/routing/not_found.rs` | ~30 | Default 404 fallback handler |

### Data structure

```
Router<S>  (Arc<RouterInner<S>>)
  └── PathRouter<S>
        ├── matchit::Router<RouteId>   ← radix tree  path → RouteId
        ├── Vec<Endpoint<S>>           ← RouteId index → MethodRouter | Route
        └── v7_checks: bool

MethodRouter<S, E>
  ├── MethodEndpoint { None | Route | BoxedHandler } × 9 methods
  ├── fallback: Fallback<S>
  └── allow_header: AllowHeader
```

### Request dispatch flow

```
Router::call()                     routing/mod.rs:614
  └── call_with_state()            routing/mod.rs:452
        └── PathRouter::call_with_state()   path_router.rs:325
              ├── matchit node.at(path)     path_router.rs:342  ← O(n) radix lookup
              ├── insert url_params         path_router.rs:353
              └── MethodRouter::call_with_state()  method_routing.rs:1167
                    └── call!(req, POST, post)      method_routing.rs:1193
```

---

## Scoped rewrite: replace the path matcher

The highest-value, lowest-risk rewrite: **abstract the path matching** so
implementations can be swapped without touching the public API.

### Step 1 — Introduce a `RouteMatcher` trait

**New file:** `axum/src/routing/matcher.rs`

```rust
pub(crate) trait RouteMatcher: Send + Sync + 'static {
    fn insert(&mut self, path: &str, id: RouteId) -> Result<(), String>;
    fn at<'p>(&self, path: &'p str) -> Option<MatchResult<'p>>;
}

pub(crate) struct MatchResult<'p> {
    pub route_id: RouteId,
    pub params: Params<'p>,          // iterator of (name, value) pairs
}
```

### Step 2 — Wrap `matchit` behind the trait (zero behaviour change)

**New file:** `axum/src/routing/matcher/matchit_impl.rs`

```rust
pub(crate) struct MatchitMatcher {
    router: matchit::Router<RouteId>,
}

impl RouteMatcher for MatchitMatcher {
    fn insert(&mut self, path: &str, id: RouteId) -> Result<(), String> {
        self.router.insert(path, id).map_err(|e| e.to_string())
    }

    fn at<'p>(&self, path: &'p str) -> Option<MatchResult<'p>> {
        self.router.at(path).ok().map(|m| MatchResult {
            route_id: *m.value,
            params: m.params.into(),
        })
    }
}
```

### Step 3 — Update `PathRouter` to use the trait

**Modified:** `axum/src/routing/path_router.rs`

```rust
pub(super) struct PathRouter<S> {
    routes: Vec<Endpoint<S>>,
-   node: Arc<Node>,           // matchit-specific Node type
+   matcher: Arc<dyn RouteMatcher>,
    v7_checks: bool,
}
```

`call_with_state` (`path_router.rs:342`) becomes:

```rust
- match self.node.at(parts.uri.path()) {
+ match self.matcher.at(parts.uri.path()) {
      Some(m) => {
          url_params::insert_url_params(&mut parts.extensions, &m.params);
          …
      }
      None => Err((Request::from_parts(parts, body), state)),
  }
```

### Step 4 — Implement the new matcher

**New file:** `axum/src/routing/matcher/custom.rs`

Implement any algorithm (hash-trie, DFA, regex-based) behind the same trait.
The rest of the framework is unaffected.

### Step 5 — Verify

```bash
cargo test --workspace          # all 635 tests must pass
cargo bench                     # compare routing throughput before/after
```

---

## Files changed

| File | Change |
|---|---|
| `axum/src/routing/matcher.rs` | **New** — `RouteMatcher` trait |
| `axum/src/routing/matcher/matchit_impl.rs` | **New** — existing matchit wrapped |
| `axum/src/routing/matcher/custom.rs` | **New** — replacement implementation |
| `axum/src/routing/path_router.rs` | **Modified** — use `dyn RouteMatcher` |
| All other routing files | **Unchanged** |
| Public API (`Router<S>`, extractors, handlers) | **Unchanged** |

---

## What must NOT be rewritten without a separate RFC

| Component | Why |
|---|---|
| `Router<S>` public API | Breaking change for every axum user |
| `url_params` extension contract | Shared between `PathRouter` and `Path<T>` extractor |
| `MethodRouter` method dispatch | Stable, no known issues |
| State via `Request::extensions` | Changing this breaks `State<T>` |
| `Route` / `RouteFuture` types | Tightly coupled to Tower `Service` trait |

---

## Risk matrix

| Risk | Likelihood | Mitigation |
|---|---|---|
| Path matching regression | Medium | 635 existing tests + property tests with `proptest` |
| Performance regression | Low–Medium | Benchmark with `criterion` before merging |
| Nested route / prefix breakage | Low | `routing/tests/nest.rs` covers these cases |
| Wildcard capture breakage | Low | `routing/tests/mod.rs` has wildcard tests |
| Breaking `matched-path` feature | Low | Covered by `extract::matched_path` tests |
