# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A study and analysis project for the [axum](https://github.com/tokio-rs/axum) Rust web framework. There is no application source code here — the repository is a structured collection of deep-dive analyses, architecture documents, request-trace walkthroughs, and code-quality reports produced by examining the axum codebase directly.

The axum source being analysed lives at `https://github.com/tokio-rs/axum.git` and is **not** checked out locally. Fetch source files via `gh api` when needed:

```bash
gh api repos/tokio-rs/axum/contents/<path> --jq '.content' | base64 -d
```

## Repository layout

| Path | Contents |
|------|----------|
| `docs/` | Reference documents: architecture overview, request lifecycle diagrams, entry-point catalogue, dependency tables, quality report |
| `docs/diagrams/` | Mermaid diagrams (see table below) and their rendered PNG/HTML previews |
| `answers/` | Question-driven analysis files — one file per prompt/question |
| `answers/improved answers/` | Revised or expanded versions of earlier answers |
| `answers/answer for last subpart/` | Numbered answer variants for the final sub-question |

### Diagram inventory

| File | Diagram type | What it shows |
|------|-------------|---------------|
| `docs/diagrams/architecture.md` | Flowchart | System layers: TCP → serve → hyper → Tower middleware → Router → Handler → extractors/responses; crate boundaries |
| `docs/diagrams/request-sequence.md` | Flowchart (detailed) | Full happy-path + error paths with file:line annotations at every node |
| `docs/diagrams/data_models.md` | ER diagram | Key types and traits as entities (`Router`, `PathRouter`, `MethodRouter`, `Request`, `Response`, trait relationships) |
| `docs/diagrams/sequence_request_lifecycle.md` | **Sequence diagram** | Autonumbered actor-to-actor message flow for `POST /users`; `alt`/`loop` blocks for 404, 405, and extraction failures |
| `docs/diagrams/state_router_and_request.md` | **State diagram (×2)** | (1) `Router<S>` compile-time type-state machine: Building → Ready → Serving; (2) runtime request processing states: Accepted → PathMatching → … → Sending |
| `docs/diagrams/class_type_hierarchy.md` | **Class diagram (×2)** | (1) Routing ownership chain: `Router` → `RouterInner` → `PathRouter` → `Node` → `Endpoint` → `MethodRouter` → `MethodEndpoint`; (2) trait system: `Service`, `Handler`, `FromRequestParts`, `FromRequest`, `IntoResponse` with all concrete implementors |

## Document conventions

- **`docs/`** files are stable reference material; update them when the analysis changes.
- **`answers/`** files are named after the prompt they address (e.g. `routing_analysis.md`, `fix_the_bug.md`). New answers go here.
- Numbered prefixes (`01_`, `02_`) in `answers/answer for last subpart/` indicate ordering within a multi-part answer set.
- When writing a new answer that supersedes an old one, place it in `answers/improved answers/` rather than overwriting.

## axum architecture quick reference

The analysed codebase is a **Cargo workspace** of four crates:

```
axum-macros  →  axum-core  →  axum  →  axum-extra
```

- **`axum-core`** — stable trait definitions: `FromRequest`, `FromRequestParts`, `IntoResponse`, `Body`. Library authors depend on this.
- **`axum`** — the main crate: `Router<S>`, `PathRouter` (matchit radix tree), `MethodRouter`, all built-in extractors, `axum::serve()`.
- **`axum-macros`** — proc-macros only (`#[debug_handler]`, `#[derive(FromRequest)]`). No dependency on `axum`.
- **`axum-extra`** — optional, feature-gated extras: typed cookies, `TypedPath`, Protobuf, JSON Lines.

### Key source files in `axum/src/`

| File | Role |
|------|------|
| `routing/mod.rs` | `Router<S>` builder, Tower `Service` impl, `call_with_state`, fallback wiring |
| `routing/path_router.rs` | `PathRouter` owns the matchit tree + `RouteId`→endpoint `Vec`; inserts `UrlParams`/`MatchedPath`/`OriginalUri` into extensions |
| `routing/method_routing.rs` | `MethodRouter` dispatches by HTTP method via the `call!` macro; builds `Allow` header; HEAD auto-falls through to GET |
| `routing/route.rs` | `Route` — type-erased Tower `Service` for a single endpoint |
| `handler/mod.rs` | `Handler` trait + `impl_handler!` macro generating blanket impls for async fns of arity 0–16 |
| `extract/path.rs` | `Path<T>` — reads `UrlParams` from `Request::extensions`, deserialises via serde |
| `extract/state.rs` | `State<T>` — reads app state inserted by `.with_state()` from extensions |
| `json.rs` | `Json<T>` extractor (validates `Content-Type`, buffers body, `serde_json` deserialise) and response |
| `serve/mod.rs` | Accept loop, per-connection Tokio task spawning, graceful shutdown |

### Request flow (summary)

```
TcpListener::accept()
  → axum::serve / handle_connection  [serve/mod.rs]
  → hyper HTTP framing
  → Router::call()                   [routing/mod.rs]
  → PathRouter::call_with_state()    [routing/path_router.rs]
      matchit.at(path) → RouteId
      insert UrlParams / MatchedPath into extensions
  → MethodRouter::call_with_state()  [routing/method_routing.rs]
      call! macro → select handler by HTTP method
      BoxedHandler::into_route(state)
  → Handler::call()                  [handler/mod.rs]
      FromRequestParts extractors (State, Path, Query, …)
      FromRequest extractor (Json, Form, Bytes — consumes body, must be last)
      user async fn → IntoResponse
  → Response → hyper → socket write
```

### Extractor contract

- **`FromRequestParts`** — cannot consume the body; runs on `&mut Parts`. Use for `Path`, `Query`, `State`, `Extension`, `MatchedPath`, headers.
- **`FromRequest`** — may consume the body; must be the **last** parameter in a handler. Use for `Json<T>`, `Form<T>`, `Bytes`, `String`, `Multipart`.

### State type-state invariant

`Router<S>` is generic over *missing* state `S`. Only `Router<()>` (state fully provided via `.with_state(s)`) satisfies the `Service` impl and can be passed to `axum::serve()`. This is enforced at compile time — not runtime.

## Key known issues / refactoring notes (from `answers/routing_analysis.md`)

- `mod.rs:392` has a `// TODO make this better.` comment — the fallback-layer closure in `fallback_endpoint` is duplicated for the `"/"` and `FALLBACK_PARAM_PATH` registrations.
- `method_routing.rs:1053–1066` has a manual 9-field `is_none()` conjunction in `route_layer`; should be extracted to `has_routes()` to mirror `PathRouter`.
