# axum — Extractor Decision Flowchart

How to choose between `FromRequestParts` and `FromRequest` when writing a handler,
and what axum's `impl_handler!` macro enforces at compile and runtime.

---

## Decision Tree

```mermaid
flowchart TD
    START(["You are adding a parameter\nto an axum handler fn"])

    Q1{"Does this extractor\nneed to read the request body?\n(buffer, stream, or parse it)"}

    START --> Q1

    Q1 -->|"No — reads headers,\npath captures, query string,\nextensions, or app state"| FRP

    Q1 -->|"Yes — needs to consume\nor buffer the body bytes"| FR

    subgraph FRP_BOX["FromRequestParts  ·  axum-core/src/extract/mod.rs"]
        FRP["Implement\nFromRequestParts&lt;S&gt;"]
        FRP_SIG["fn from_request_parts\n  (parts: &mut Parts,\n   state: &S)\n  → Result&lt;Self, Rejection&gt;"]
        FRP_RULES["✅  Any number per handler\n✅  Any position in the arg list\n✅  Runs before the body is touched\n✅  Multiple can borrow parts in sequence"]
        FRP_BUILTINS["Built-ins:\nPath&lt;T&gt;  ·  Query&lt;T&gt;  ·  State&lt;T&gt;\nExtension&lt;T&gt;  ·  Method  ·  Uri\nHeaderMap  ·  MatchedPath  ·  OriginalUri"]
        FRP --> FRP_SIG
        FRP_SIG --> FRP_RULES
        FRP_RULES --> FRP_BUILTINS
    end

    subgraph FR_BOX["FromRequest  ·  axum-core/src/extract/mod.rs"]
        FR["Implement\nFromRequest&lt;S&gt;"]
        FR_SIG["fn from_request\n  (req: Request,\n   state: &S)\n  → Result&lt;Self, Rejection&gt;"]
        FR_RULES["⚠️  Exactly ONE per handler\n⚠️  Must be the LAST parameter\n⚠️  Consumes the entire Request\n     (body stream + parts)"]
        FR_BUILTINS["Built-ins:\nJson&lt;T&gt;  ·  Form&lt;T&gt;  ·  Bytes\nString  ·  Multipart  ·  RawBody"]
        FR --> FR_SIG
        FR_SIG --> FR_RULES
        FR_RULES --> FR_BUILTINS
    end

    FRP_BOX --> POSITION{"Is the FromRequest\nextractor the last\nparameter?"}
    FR_BOX  --> POSITION

    POSITION -->|"Yes — or there is\nno FromRequest extractor"| VALID

    POSITION -->|"No — a FromRequest\nextractor appears before\nanother parameter"| COMPILE_ERR

    COMPILE_ERR["❌  Compile error\nimpl_handler! macro generates impls\nfor arity 0–16 with FromRequest\nalways at the final position.\nMismatched arity = no impl found."]

    VALID["✅  Valid handler signature"]

    VALID --> RUNTIME

    subgraph RUNTIME["Runtime execution order — handler/mod.rs impl_handler! (L238)"]
        direction TB
        R1["1. req.into_parts() → (parts, body)"]
        R2["2. Loop: from_request_parts(&mut parts, &state)\n   for each FromRequestParts arg\n   short-circuits on first Err(rejection)"]
        R3["3. Request::from_parts(parts, body)\n   reconstruct the full request"]
        R4["4. from_request(req, &state)\n   for the FromRequest arg (if any)"]
        R5["5. Call user async fn with all extracted values\n   → return type must implement IntoResponse"]
        R1 --> R2 --> R3 --> R4 --> R5
    end

    RUNTIME --> OUTCOME{"Extractor\nresult?"}

    OUTCOME -->|"All Ok(value)"| CALL["User fn runs\nResponse returned to hyper"]
    OUTCOME -->|"Any Err(rejection)"| REJECT["rejection.into_response()\n400 Bad Request / 415 Unsupported\n422 Unprocessable / 404 Not Found\n(depends on extractor)"]
```

---

## Why the ordering rule exists

`FromRequestParts` takes `&mut Parts` — the borrow checker allows multiple
sequential borrows as long as they don't overlap. The `impl_handler!` macro
generates code that borrows `parts` once per extractor, in declaration order.

`FromRequest` takes ownership of the entire `Request` (which includes the body
`Body` stream). There can only be one owner, and body bytes are consumed
non-reversibly — so only one such extractor can exist, and it must run last
after all `FromRequestParts` extractors are done with `parts`.

## Rejection type contract

Every extractor must declare:

```rust
type Rejection: IntoResponse;
```

This guarantees that any extraction failure can be converted to a valid HTTP
response automatically — no `unwrap`, no panic, no special error-handling glue
in the handler body.

## Custom extractor checklist

| Step | What to do |
|------|-----------|
| 1 | Decide: does it need the body? → choose trait |
| 2 | Define a `Rejection` type that `impl IntoResponse` |
| 3 | `impl FromRequestParts<S>` or `impl FromRequest<S>` |
| 4 | Place the extractor at the correct position in the handler |
| 5 | Optionally add `#[derive(FromRequest)]` via `axum-macros` for struct extractors |
