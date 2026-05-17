# Exploration Log — axum Codebase Study Session

**Date:** 2026-05-17  
**Subject:** [tokio-rs/axum](https://github.com/tokio-rs/axum) — Rust web framework  
**Method:** Remote source reading via `gh api repos/tokio-rs/axum/contents/<path>` (no local checkout)

---

## 20:10 — Initial Reconnaissance

**Prompt:** _Explore the axum repository. What is it? How is it structured?_

**Actions:**
- Fetched `Cargo.toml` (workspace manifest) to discover the four-crate layout
- Fetched `axum/src/lib.rs`, `axum-core/src/lib.rs`, `axum-extra/src/lib.rs`, `axum-macros/src/lib.rs`
- Walked top-level directories: `examples/`, `contrib/`, `deny.toml`, `.clippy.toml`

**Key findings:**
- Cargo workspace of **four published crates** in strict dependency order: `axum-macros → axum-core → axum → axum-extra`
- `axum-macros` has **no runtime dependency on axum** — proc-macro only (`syn`, `quote`, `proc-macro2`)
- `axum-core` is versioned independently so library authors can depend on it without coupling to axum's release cycle
- Workspace-wide `forbid(unsafe_code)` lint — zero unsafe Rust anywhere
- `deny.toml` enforces licence allow-lists and bans duplicate dependencies via `cargo-deny`

**Produced:** `docs/exploration-log/top_level_directories.md`

---

## 20:20 — Code Quality Audit

**Prompt:** _Run the test suite. What is the test coverage and quality?_

**Actions:**
- Ran `cargo test --workspace` against a local clone in `/tmp/axum/`
- Inspected Clippy config, `deny.toml`, doc-test structure

**Key findings:**
- **614 tests pass**, 0 fail, 1 ignored across all crates
- `axum` unit + integration tests: 367 passed in 0.14 s
- `axum` doc-tests: 165 passed in 38.80 s (every code example in docs is compiled and run)
- `axum-core`: 19 tests; `axum-extra`: 27 tests; `axum-macros`: 36 tests (compile-time UI tests via `trybuild`)
- `trybuild` tests in `axum-macros` compile-fail snippets to verify that `#[debug_handler]` produces the correct error messages

**Produced:** `docs/exploration-log/quality_report.md`

---

## 20:28 — Architecture Deep-Dive

**Prompt:** _Describe the full architecture. How does a request travel from TCP socket to user handler?_

**Actions:**
- Read `axum/src/serve/mod.rs`, `routing/mod.rs`, `routing/path_router.rs`, `routing/method_routing.rs`, `handler/mod.rs`
- Traced `axum::serve()` → `hyper` → `Router::call()` → `PathRouter` → `MethodRouter` → `Handler::call()` → extractors → user `async fn`
- Catalogued all external dependencies with versions and roles

**Key findings:**
- `axum::serve()` spawns one **Tokio task per TCP connection** (backpressure = OS TCP backlog)
- `Router<S>` wraps `Arc<RouterInner<S>>` — cloning is O(1) (single atomic increment), enabling zero-copy per-connection router dispatch
- `PathRouter` owns both a **matchit radix tree** (`Node`) and a `Vec<Endpoint<S>>` indexed by `RouteId(usize)` — the Vec index **is** the `RouteId`
- `MethodRouter` dispatches via the `call!` macro, which expands to a chain of `if req.method() == Method::X` — intentionally exhaustive so a new method verb can't be silently missed
- `BoxedHandler` stores handlers before `.with_state()` is called; `into_route(state)` finalises them — this is how the type-state `Router<S>` → `Router<()>` transition works at runtime

**Produced:** `docs/diagrams/architecture.md` (flowchart + full reference tables)

---

## 20:45 — First Diagram + HTML Preview

**Prompt:** _Generate a request flow diagram with file:line annotations._

**Actions:**
- Wrote detailed `flowchart TD` Mermaid diagram annotating every node with source file and line number
- Generated rendered HTML preview via Mermaid CDN

**Key findings:**
- Identified the exact call chain with file:line: `serve/mod.rs:559` → `path_router.rs:342` → `method_routing.rs:1193` → `handler/mod.rs:238`
- Confirmed HEAD falls through to GET automatically in `MethodRouter` (saves users from registering HEAD separately)

**Produced:** `docs/diagrams/request-sequence.md`, `docs/visualisation/preview-request-sequence.html`

---

## 20:57 — Bug Analysis

**Prompt:** _Fix the bug._ (referring to known issues in the codebase)

**Actions:**
- Located `// TODO make this better.` at `routing/mod.rs:392`
- Read `fallback_endpoint()` — the `layer_fn` closure is copy-pasted verbatim for `"/"` and `FALLBACK_PARAM_PATH` registrations
- Located `method_routing.rs:1053–1066` — manual 9-field `is_none()` conjunction in `route_layer`

**Key findings:**
- **Bug 1 (mod.rs:392):** The two identical closures in `fallback_endpoint` should be extracted into a local `fn make_fallback_layer(endpoint)` to eliminate the duplication
- **Bug 2 (method_routing.rs:1053):** The 9-field `is_none()` chain (`self.get.is_none() && self.post.is_none() && …`) should become a `has_routes()` method, mirroring how `PathRouter` handles the same check
- Both are low-risk refactors with no behaviour change — confirmed by reading surrounding code and test coverage

**Produced:** `docs/prompts/fix_the_bug.md`, `docs/prompts/improved answers/fix_the_bug.md`

---

## 20:58 — Routing System Rewrite Analysis

**Prompt:** _Rewrite the routing system._

**Actions:**
- Read the full `routing/` module: `mod.rs`, `path_router.rs`, `method_routing.rs`, `route.rs`
- Analysed all abstractions: `Router<S>`, `RouterInner<S>`, `PathRouter<S>`, `Node`, `MethodRouter<S,E>`, `Fallback<S>`, `Endpoint<S>`

**Key findings:**
- `map_inner!` / `tap_inner!` macros (mod.rs:129–152) exist solely to encapsulate `Arc::try_unwrap` + mutate + re-wrap — clever but hides mutation behind macro syntax
- `Node` maintains **two HashMaps** (`route_id_to_path`, `path_to_route_id`) because matchit has no "does this path exist?" API — the second HashMap fills that gap for same-path method merging
- `Fallback<S>` has three variants: `Default` (built-in 404), `Service`, `BoxedHandler` — `merge()` panics if both sides carry a non-`Default` fallback (by design, enforced at build time)
- The `call!` macro destructures all 9 method fields explicitly — ensures no new HTTP verb can be added without touching dispatch

**Produced:** `docs/prompts/routing_analysis.md`, `docs/prompts/improved answers/rewrite_routing_system.md`

---

## 21:00 — Caching Design

**Prompt:** _Add caching using FastRustCache._

**Actions:**
- Analysed where caching could be injected into the Tower middleware stack
- Designed a Tower `Layer` / `Service` wrapper approach using `axum::middleware::from_fn`
- Verified the extractor + state patterns needed for cache key construction

**Key findings:**
- Correct insertion point is `.route_layer(CacheLayer::new(...))` — not `.layer()` — so cache only applies to matched routes, not 404 paths
- Cache key must be built from `MatchedPath` + query string + optionally `Authorization` header; `MatchedPath` is only available after `PathRouter` runs, confirming `.route_layer()` placement
- `State<CacheHandle>` is the idiomatic way to share the cache handle across handlers

**Produced:** `docs/prompts/add_caching_fastrustcache.md`, `docs/prompts/improved answers/caching_verification_and_design.md`

---

## 21:07 — Path Extractor Bug

**Prompt:** _There's a bug in Path<T> extraction._

**Actions:**
- Read `axum/src/extract/path/mod.rs` in full
- Traced `UrlParams` insertion in `path_router.rs` → `url_params::insert_url_params()` → `Path::from_request_parts()`
- Checked percent-encoding handling and serde deserialisation error path

**Key findings:**
- `Path<T>` reads from `Request::extensions()`, not from the URI directly — if `UrlParams` is missing (e.g. wrong router setup), it returns a `MissingPathParams` rejection, not a panic
- Percent-encoded captures are decoded before serde sees them (`percent_encoding::percent_decode_str`)
- The serde rejection wraps `serde_path_to_error` output — users get the **field path** that failed, not just "deserialization failed"

**Produced:** `docs/prompts/path_extractor_bug.md`

---

## 21:43 — Code Review

**Prompt:** _Code review._

**Actions:**
- Reviewed all files produced in the session for accuracy, completeness, consistency
- Cross-checked file:line citations against source
- Audited the architecture diagram for missing edges

**Key findings:**
- All file:line references verified correct against the fetched source
- `entry_points_and_dependencies.md` was the most complete standalone reference document produced
- Identified gap: no diagram covering the extractor decision logic (added later at 22:52)

**Produced:** `docs/0-bug-audit.md`, `docs/bug-audit.md`, `docs/prompts/improved answers/` (all three)

---

## 22:01 — Context Management Task

**Prompt:** _Axum request lifecycle — explain as a step-by-step trace for `POST /users`._

**Actions:**
- Wrote two variants: a short summary (`01_request_lifecycle.md`) and a full annotated trace (`02_axum_request_lifecycle.md`)
- Focused on the exact moment `impl_handler!` splits Request into `(parts, body)`

**Key findings:**
- The split at `req.into_parts()` is the boundary between `FromRequestParts` and `FromRequest` — once parts are split off, the body stream is still attached to `body`, which is only handed to `FromRequest` after all parts extractors complete
- This is why `FromRequest` must be last: there is literally only one `body` value, and it gets moved into the extractor

**Produced:** `docs/contex_management/01_request_lifecycle.md`, `docs/contex_management/02_axum_request_lifecycle.md`

---

## 22:34 — CLAUDE.md Bootstrapped

**Prompt:** (implicit — session housekeeping)

**Actions:**
- Created `CLAUDE.md` with full architecture quick-reference, extractor contract, key source files table, request flow summary, and known issues section

**Produced:** `docs/CLAUDE.md`

---

## 22:39 — Three New Mermaid Diagrams (Sequence · State · Class)

**Prompt:** _Add sequence diagram, state diagram, class diagram._

**Key findings / design decisions:**
- **Sequence diagram** (`sequence_request_lifecycle.md`): used `autonumber` + `alt`/`loop` blocks to show all four outcomes (200, 404, 405, 4xx rejection) in one diagram without branching into separate files
- **State diagrams** (`state_router_and_request.md`): two machines in one file — compile-time type-state of `Router<S>` and runtime request-processing state; the compile-time diagram has no equivalent in other frameworks and is axum's most distinctive design
- **Class diagrams** (`class_type_hierarchy.md`): two diagrams — routing ownership chain (Arc nesting) and trait system; annotated why `BoxedHandler` variant exists (state not known at registration time)

**Produced:** `docs/last_diagrams/sequence_request_lifecycle.md`, `docs/last_diagrams/state_router_and_request.md`, `docs/last_diagrams/class_type_hierarchy.md`

---

## 22:52 — Three More Mermaid Diagrams (Extractor · Middleware · Feature Flags)

**Prompt:** _Create at least 3 different Mermaid diagrams._

**Key findings / design decisions:**
- **Extractor decision flowchart** (`extractor_decision.md`): the `FromRequestParts` vs `FromRequest` choice is the single most common point of confusion for new axum users — making it a flowchart with the runtime execution order explicitly shown addresses this
- **Middleware composition** (`middleware_composition.md`): the counter-intuitive fact that the **last** `.layer()` call is the **outermost** layer (first to see a request) is not obvious from the API; the "onion" diagram + `.layer()` vs `.route_layer()` side-by-side makes it concrete
- **Feature flags** (`feature_flags.md`): four sub-diagrams (one per crate) make the Cargo feature surface navigable; included a "minimal binary vs full-featured" tradeoff flowchart at the end

**Produced:** `docs/last_diagrams/extractor_decision.md`, `docs/last_diagrams/middleware_composition.md`, `docs/last_diagrams/feature_flags.md`

---

## 23:04 — Interactive D3.js Visualization (online)

**Prompt:** _Generate an interactive D3.js or HTML visualization based on the whole project treating as a story._

**Design:** Four-chapter narrative told through a single browser page:

| Chapter | Type | Key interaction |
|---------|------|-----------------|
| 01 Crate Architecture | Force-directed graph | Drag nodes to explore dependency structure |
| 02 Request Journey | Animated flow | ▶ buttons for Happy Path / 404 / 405 / Extractor fail |
| 03 Trait System | Force-directed graph | Drag, click trait → see signature and role |
| 04 Error Paths | Decision tree | Click any node → see what triggers it and the HTTP status |

**Technical:** D3.js v7 from CDN; `window.addEventListener('load', () => requestAnimationFrame(render))` to ensure layout is settled before computing dimensions.

**Produced:** `docs/last_diagrams/axum_story_interactive.html`

---

## 23:21 — Offline HTML Visualization (no internet required)

**Prompt:** _Add a second HTML that works without internet._

**Design:** Identical four chapters, zero external dependencies:
- **Force simulation** written from scratch in ~60 lines of vanilla JS: repulsion between all node pairs, spring forces on links, weak centre gravity, velocity damping — converges in ~150–200 steps
- **Packet animation** via `requestAnimationFrame` with linear interpolation between waypoints; speed constant in px/s so longer segments take proportionally longer
- **Drag** via `mousedown/mousemove/mouseup` on SVG using `getBoundingClientRect()` for coordinate mapping
- **Labels in Polish** — descriptions translated for accessibility
- SVG elements created with `document.createElementNS(NS, tag)` — no wrapper libraries

**Produced:** `docs/last_diagrams/axum_story_offline.html`

---

## 23:35 — Repository Reorganisation + Push

**Prompt:** _Update GitHub repo with these files and structure._

**Restructuring applied:**

| Old path | New path |
|----------|----------|
| `CLAUDE.md` | `docs/CLAUDE.md` |
| `answers/` | `docs/prompts/` |
| `answers/improved answers/` | `docs/prompts/improved answers/` |
| `answers/answer for last subpart/` | `docs/contex_management/` |
| `docs/quality_report.md` | `docs/exploration-log/` |
| `docs/request_trace.md` | `docs/exploration-log/` |
| `docs/top_level_directories.md` | `docs/exploration-log/` |
| `docs/diagrams/preview-*.{html,png}` | `docs/visualisation/` |
| `docs/diagrams/{class,sequence,state}*.md` | `docs/last_diagrams/` |
| *(new)* | `docs/last_diagrams/extractor_decision.md` |
| *(new)* | `docs/last_diagrams/middleware_composition.md` |
| *(new)* | `docs/last_diagrams/feature_flags.md` |
| *(new)* | `docs/last_diagrams/axum_story_interactive.html` |
| *(new)* | `docs/last_diagrams/axum_story_offline.html` |

**Commit:** `62bcbd8` — pushed to `kaszynskaa/axum-analysis` (branch `main`)

---

## Session Summary

| Metric | Value |
|--------|-------|
| Duration | ~3 h 25 min (20:10 → 23:35) |
| Commits | 24 |
| Files produced | 38 |
| axum source files read | ~20 (remote via `gh api`) |
| Diagrams created | 9 Mermaid + 2 interactive HTML |
| Biggest single finding | The `Router<S>` type-state machine — compile-time enforcement of "state must be provided before serving" is unique to axum and has no runtime cost |
| Trickiest detail | `BoxedHandler` exists because handlers are registered before `.with_state()` is called; the type `S` is not yet `()` at registration time, so handlers must be stored in a type-erased box until `into_route(state)` finalises them |
