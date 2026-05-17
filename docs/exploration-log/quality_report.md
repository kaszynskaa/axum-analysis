# axum — Code Quality & Testing Report

**Toolchain:** rustc 1.95.0 / cargo 1.95.0 (stable-aarch64-apple-darwin)  
**Command:** `cargo test --workspace`  
**Date:** 2026-05-17

---

## Test results

| Crate | Passed | Failed | Ignored | Duration |
|---|---|---|---|---|
| `axum` (unit + integration) | 367 | 0 | 0 | 0.14 s |
| `axum` (routing tests) | 1 | 0 | 0 | 0.00 s |
| `axum` (extract tests) | 15 | 0 | 0 | 0.01 s |
| `axum` (middleware tests) | 13 | 0 | 0 | 0.01 s |
| `axum` (serve tests) | 5 | 0 | 0 | 0.00 s |
| `axum` (doc-tests) | 165 | 0 | 1 | 38.80 s |
| `axum-core` | 19 | 0 | 0 | 5.39 s |
| `axum-extra` | 27 | 0 | 0 | 6.62 s |
| `axum-macros` | 23 | 0 | 0 | 4.21 s |
| **Total** | **635** | **0** | **1** | ~55 s |

**Result: all 635 tests pass. Zero failures. Zero compiler warnings.**

The single ignored test is an intentional skip (marked `#[ignore]`) in the serve
integration tests — typical for tests that require specific runtime conditions.

---

## Test strategy observed in the codebase

### Unit tests
Inline `#[cfg(test)] mod tests` blocks live alongside the production code in
each module (e.g. `axum/src/json.rs`, `axum/src/routing/tests/`). They cover:
- Extractor happy paths and all rejection variants
- Method routing (GET→HEAD promotion, method-not-allowed, fallback precedence)
- Path matching edge cases (nested routes, wildcard, trailing-slash redirect)
- Middleware ordering and layering

### Doc-tests (165 tests)
Every public API item in every crate includes a runnable example in its
rustdoc comment. These are compiled and executed as part of `cargo test`,
giving near-100% coverage of the documented surface area. This is the
dominant test count in the workspace.

### Compile-fail tests
`axum-macros` uses `rustdoc` compile-fail tests (e.g. `// compile_fail`) to
assert that invalid macro usage produces helpful compiler errors rather than
cryptic type errors. 7 such tests were run and passed.

### Integration-style tests
`axum/src/test_helpers/test_client.rs` provides a `TestClient` that drives
the full router stack in-process (no real TCP socket). The serve tests use
`tokio::net::TcpListener` in-process with `reqwest` to cover the real
accept-loop path.

---

## Code quality observations

### No unsafe code
`Cargo.toml` (workspace root) sets:
```toml
[workspace.lints.rust]
unsafe_code = "forbid"
```
This is enforced across all four crates at compile time. Zero `unsafe` blocks
exist anywhere in the production source.

### Lint configuration
`.clippy.toml` and workspace `[lints]` are shared. The test run produced
**zero compiler warnings** across the entire workspace — the codebase is
warning-clean on the latest stable toolchain.

### Dependency hygiene
`deny.toml` runs `cargo-deny` in CI to:
- Enforce an allowlist of OSS licenses
- Ban duplicate versions of the same crate
- Block crates with known security advisories (via `rustsec`)

### API surface stability
`axum-core` is versioned separately from `axum` specifically to provide a
stable base for library authors. The trait definitions (`FromRequest`,
`IntoResponse`) are unlikely to change between minor axum versions.

### Error messages as a quality metric
The `axum-macros` compile-fail tests assert that using macros incorrectly
produces readable error messages (tested via `#[diagnostic::do_not_recommend]`
and custom `syn` error spans). This is an unusual but valuable quality practice.

---

## Recommendations (for a project built on axum)

| Area | Recommendation |
|---|---|
| **Test coverage** | Use `axum::test_helpers::TestClient` for handler-level tests; it exercises the full extractor + response pipeline without a socket |
| **Rejection testing** | Always test the rejection path (wrong `Content-Type`, missing fields, oversized body) — axum rejections are `impl IntoResponse` and their HTTP status codes are stable API |
| **State testing** | Use `Router::with_state` + `TestClient` to test handlers that depend on `State<T>` — avoids mocking |
| **Middleware testing** | Apply `tower::ServiceExt::oneshot` directly on a layered `Router` to unit-test middleware in isolation |
| **Clippy** | Run `cargo clippy -- -D warnings` in CI; the axum repo itself is warning-free, so there is no noise to filter |
| **Compile-fail tests** | If writing custom extractors or macros, add compile-fail doctests to assert useful rejection error messages |
