# axum — Middleware Composition & Execution Order

How Tower layers stack around axum's `Router`, in what order they execute,
and how `.layer()` differs from `.route_layer()`.

---

## The Onion Model

Each call to `.layer(L)` wraps the current service in a new outer layer.
**The last `.layer()` call is the outermost layer — the first to see a request.**

```mermaid
flowchart TD
    INCOMING["Incoming HTTP Request\n(from hyper / TcpStream)"]

    subgraph STACK["Tower Middleware Stack — outermost first"]
        direction TB

        subgraph L3["Outermost Layer\n.layer(TraceLayer)  ← registered LAST"]
            subgraph L2["Middle Layer\n.layer(CorsLayer)"]
                subgraph L1["Innermost .layer()\n.layer(CompressionLayer)  ← registered FIRST"]
                    subgraph RL["Route-only layers\n.route_layer(AuthLayer)"]
                        CORE["Router core\nPathRouter → matchit lookup\nMethodRouter → method dispatch\nHandler → extractor chain → user fn"]
                    end
                end
            end
        end
    end

    RESP["HTTP Response\n(back to hyper)"]

    INCOMING -->|"① TraceLayer pre-request"| L3
    L3       -->|"② CorsLayer pre-flight check"| L2
    L2       -->|"③ CompressionLayer wraps response stream"| L1
    L1       -->|"④ AuthLayer validates token\n(only for matched routes)"| RL
    RL       -->|"⑤ Router dispatches to handler"| CORE
    CORE     -->|"⑥ Handler runs, response produced"| RL
    RL       -->|"⑦ AuthLayer post-processing"| L1
    L1       -->|"⑧ CompressionLayer compresses body"| L2
    L2       -->|"⑨ CorsLayer adds response headers"| L3
    L3       -->|"⑩ TraceLayer records span + status"| RESP
```

---

## `.layer()` vs `.route_layer()` — the critical difference

```mermaid
flowchart LR
    subgraph DOTLAYER["router.layer(L)"]
        direction TB
        LA_REQ["Request arrives"]
        LA_MW["L runs\n(even for 404 / fallback paths)"]
        LA_ROUTER["Router dispatches\n(matched or fallback)"]
        LA_RESP["Response returns through L"]
        LA_REQ --> LA_MW --> LA_ROUTER --> LA_RESP
    end

    subgraph ROUTELAYER["router.route_layer(L)"]
        direction TB
        RL_REQ["Request arrives"]
        RL_ROUTER["Router dispatches"]
        RL_MATCH{Path matched?}
        RL_MW["L runs\n(only for matched routes)"]
        RL_HANDLER["Handler executes"]
        RL_FB["Fallback (404)\nL is bypassed entirely"]
        RL_REQ --> RL_ROUTER --> RL_MATCH
        RL_MATCH -->|"Yes"| RL_MW --> RL_HANDLER
        RL_MATCH -->|"No"| RL_FB
    end
```

**Key consequence:** An authentication layer added via `.route_layer()` is
never called for 404 responses, so attackers cannot probe your route structure
through auth failures. An auth layer added via `.layer()` runs on every request
including 404s — useful if you want to hide even the existence of unmatched paths.

---

## Registration order → execution order

```mermaid
flowchart LR
    CODE["Router::new()\n  .route(...)\n  .layer(A)       ← registered first\n  .layer(B)       ← registered second\n  .layer(C)       ← registered last"]

    EXEC["Request execution order:\nC  →  B  →  A  →  Router\n\nResponse execution order:\nA  →  B  →  C"]

    CODE --> EXEC
```

`axum` builds the layer stack by calling `ServiceBuilder`-style wrapping:
each new `.layer(L)` call wraps the existing service, so later registrations
end up on the outside.

---

## Built-in `tower-http` layers and where they fit

| Layer | Typical placement | Why |
|-------|-------------------|-----|
| `TraceLayer` | Outermost (`.layer()` last) | Must observe the final HTTP status code, including 404/405 |
| `CorsLayer` | Near outermost (`.layer()`) | Must intercept OPTIONS preflight before routing |
| `CompressionLayer` | Middle (`.layer()`) | Compresses response body regardless of route |
| `TimeoutLayer` | Middle (`.layer()`) | Enforces wall-clock deadline across the whole pipeline |
| `AuthLayer` (custom) | `.route_layer()` if auth is per-route | Skip for 404; use `.layer()` if all paths require auth |
| `ConcurrencyLimitLayer` | Innermost (`.route_layer()`) | Limits concurrency per matched route, not global |

---

## `from_fn` middleware — Tower layer from a plain async fn

```mermaid
flowchart TD
    subgraph FN["axum::middleware::from_fn(my_middleware)"]
        direction TB
        MW_SIG["async fn my_middleware(\n  req: Request,\n  next: Next,\n) → Response"]
        PRE["Pre-processing:\nread/modify req before next.run(req)"]
        CALL["next.run(req)  — calls the inner service"]
        POST["Post-processing:\nread/modify response"]
        MW_SIG --> PRE --> CALL --> POST
    end

    NOTE["from_fn wraps an async fn as a Tower Layer.\nThe Next handle advances the request\nto the next layer/handler in the stack.\nEquivalent to writing a full Service impl."]
    FN --- NOTE
```

---

## Key design invariants

| Invariant | Enforcement |
|-----------|-------------|
| Layers are applied in reverse registration order | Each `.layer()` call wraps the current `Router` via `layer.layer(svc)` |
| `.route_layer()` does not wrap the fallback handler | `route_layer` is stored per-`MethodRouter` and applied only after path matching |
| State must be provided via `.with_state()` before layers can be fully resolved | `BoxedHandler` stores handlers before state is known; `into_route(state)` finalises |
| Middleware errors must be converted to responses | `HandleError` layer bridges `Service::Error` → `IntoResponse` |
