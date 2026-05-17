# axum — Top-Level Directory Guide

## Repository root: `/tmp/axum/`

| Directory / File | Purpose |
|---|---|
| `axum/` | Main crate (v0.8.9). Contains the `Router`, all built-in extractors, middleware helpers, response types, WebSocket support, and the `serve()` entry point. Everything a typical application author imports. |
| `axum-core/` | Minimal foundation crate (v0.5.6) for library and middleware authors. Defines the core traits (`FromRequest`, `FromRequestParts`, `IntoResponse`) and the `Body` newtype. Kept deliberately stable — breaking changes here affect the whole ecosystem. |
| `axum-extra/` | Optional batteries (v0.12.6). Feature-gated add-ons: typed cookies, typed HTTP headers (`headers` crate), `TypedPath` compile-time route checking, Protobuf support, JSON Lines streaming, and extra routing/middleware utilities. Nothing here is needed for basic usage. |
| `axum-macros/` | Procedural macro crate (v0.5.1). Provides `#[debug_handler]` (better compile errors on handlers), `#[derive(FromRequest)]` / `#[derive(FromRequestParts)]`, and `#[derive(TypedPath)]`. Has no dependency on `axum` itself — only `syn`, `quote`, `proc-macro2`. |
| `examples/` | 50+ self-contained runnable example projects. Covers common patterns: JSON APIs, WebSockets, TLS, JWT auth, database integration (Diesel, SQLx), SSE, multipart upload, graceful shutdown, tracing, error handling, and more. |
| `contrib/` | IDE tooling contributions (e.g. editor configuration, snippets). Not part of the published library. |
| `Cargo.toml` | Workspace manifest. Declares all four crates as workspace members and sets shared dependency versions and lint rules (including the workspace-wide `forbid(unsafe_code)`). |
| `Cargo.lock` | Pinned dependency tree for reproducible builds. |
| `deny.toml` | `cargo-deny` configuration: enforces license allowlists, bans duplicate dependencies, and checks for known security advisories. |
| `.clippy.toml` | Shared Clippy linter configuration applied across the entire workspace. |
| `CONTRIBUTING.md` | Contribution guidelines: how to run tests, how to open PRs, coding standards. |
| `CHANGELOG.md` | Version history with breaking changes, additions, and fixes per release. |
| `LICENSE` | MIT license. |
| `README.md` | Symlink to `axum/README.md`. |

## Key internal layout inside `axum/src/`

| Path | Purpose |
|---|---|
| `lib.rs` | Public re-export surface — everything a user imports comes through here. |
| `routing/` | `Router<S>`, `PathRouter` (matchit-based), `MethodRouter`, `Route`, fallback logic. |
| `extract/` | All built-in extractors: `Path`, `Query`, `State`, `Extension`, `Multipart`, `WebSocketUpgrade`, `MatchedPath`, `ConnectInfo`, `OriginalUri`, `NestedPath`. |
| `handler/` | `Handler` trait and its macro-generated blanket impls for async functions of any arity. |
| `middleware/` | `from_fn`, `from_extractor`, `map_request`, `map_response` — helpers for building middleware from plain async functions. |
| `response/` | `Html`, `Redirect`, `NoContent`, Server-Sent Events (`Sse`). |
| `serve/` | `axum::serve()`, the `Serve` struct, graceful shutdown, `TcpListener` / custom listener abstractions. |
| `body/` | `Body` newtype wrapping hyper's `Incoming`; `BodyDataStream`; `to_bytes` utility. |
| `json.rs` | `Json<T>` extractor (deserialise) and response (serialise) — behind the `json` feature flag. |
| `form.rs` | `Form<T>` extractor for `application/x-www-form-urlencoded` bodies. |
| `extension.rs` | `Extension<T>` extractor and middleware. |
| `error_handling/` | `HandleError` layer — converts service errors into responses. |
| `test_helpers/` | `TestClient` for writing integration tests without spinning up a real TCP socket. |
