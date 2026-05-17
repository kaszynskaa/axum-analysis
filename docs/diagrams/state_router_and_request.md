# axum — State Diagrams

Two state machines extracted from the axum source:
1. **Router type-state** — compile-time states of `Router<S>` from construction to serving
2. **Request processing** — runtime states of an HTTP request moving through the stack

---

## 1. Router Type-State Machine

`Router<S>` uses Rust's type system as a state machine. The generic parameter `S`
represents *missing* state. Calling `.with_state(s)` is the only transition that
produces `Router<()>`, the only variant accepted by `axum::serve`.

```mermaid
stateDiagram-v2
    direction LR

    [*] --> Building : Router::new()

    state Building {
        direction TB
        Bare --> Routed       : .route(path, method_router)
        Bare --> Routed       : .route_service(path, svc)
        Routed --> Routed     : .route() / .nest() / .merge()
        Routed --> Layered    : .layer(L) — wraps ALL routes
        Layered --> Layered   : .layer()
        Layered --> Routed    : .route()
        Routed --> WithFallback  : .fallback(handler)
        Layered --> WithFallback : .fallback(handler)
        WithFallback --> WithFallback : .layer()
    }

    Building --> Ready : .with_state(s)\nRouter~S~ → Router~()~

    note right of Ready
        Only Router<()> satisfies
        Service<Request<B>> impl.
        State is Arc-cloned once
        per accepted connection.
    end note

    Ready --> Serving : axum::serve(listener, router)\nIntoMakeService clones router\nper TCP connection

    state Serving {
        direction TB
        Accepting --> Dispatching : accept() yields TcpStream
        Dispatching --> Accepting : connection closed
    }

    Serving --> [*] : graceful shutdown / signal
```

---

## 2. Request Processing State Machine

Runtime states of a single HTTP request from TCP arrival to response dispatch.
Each state maps to a method in the routing/handler stack.

```mermaid
stateDiagram-v2
    direction TB

    [*] --> Accepted : TcpListener::accept()

    Accepted --> Parsing : hyper reads raw bytes

    state Parsing {
        direction LR
        RawBytes --> HTTPRequest : parse headers + body stream
    }

    Parsing --> PathMatching : Router::call_with_state()

    state PathMatching {
        direction TB
        URILookup : matchit radix tree\nnode.at(uri.path())
        ExtensionsWrite : insert UrlParams\ninsert MatchedPath\ninsert OriginalUri
        URILookup --> ExtensionsWrite : RouteId found
    }

    PathMatching --> MethodMatching : path matched
    PathMatching --> Fallback404   : MatchError::NotFound

    state MethodMatching {
        direction TB
        MethodCheck : call! macro\nreq.method() == X
        RouteReady  : handler.into_route(state)\nBoxedHandler → Route
        MethodCheck --> RouteReady : method registered
    }

    MethodMatching --> ExtractingParts : method matched
    MethodMatching --> Rejected405     : method not registered

    state ExtractingParts {
        direction TB
        SplitParts : req.into_parts() → (parts, body)
        RunFRP     : FromRequestParts extractors\nState Path Query Extension…
        SplitParts --> RunFRP
    }

    ExtractingParts --> ExtractingBody  : all parts extractors Ok
    ExtractingParts --> RejectedExtract : any extractor Err(rejection)

    state ExtractingBody {
        direction TB
        ContentTypeCheck : validate Content-Type header
        BufferBody       : Bytes::from_request() buffers body
        Deserialise      : serde_json / serde_html_form
        ContentTypeCheck --> BufferBody  : valid
        BufferBody --> Deserialise
    }

    ExtractingBody --> Handling        : FromRequest Ok
    ExtractingBody --> RejectedExtract : FromRequest Err(rejection)

    state Handling {
        direction TB
        UserFn    : user async fn executes\nawait handler(args…)
        Converting: value.into_response()\nJson serde_json::to_writer\nHtml / Redirect / StatusCode…
        UserFn --> Converting
    }

    Handling --> Sending

    state Sending {
        direction TB
        FrameHTTP : hyper frames Response\nas HTTP/1.1 or HTTP/2 bytes
        WriteSocket : TokioIo socket write
        FrameHTTP --> WriteSocket
    }

    Fallback404    --> Sending : 404 Not Found response
    Rejected405    --> Sending : 405 Method Not Allowed + Allow header
    RejectedExtract --> Sending : 4xx rejection.into_response()

    Sending --> [*] : response delivered\nconnection keep-alive or close
```

---

## Key design invariants

| Invariant | Where enforced |
|-----------|----------------|
| `Router<S>` with `S ≠ ()` cannot be passed to `serve()` | Rust type-checker — `Service` impl only exists for `Router<()>` |
| `FromRequestParts` extractors always run before `FromRequest` | `impl_handler!` macro in `handler/mod.rs` — parts loop before body extraction |
| Every rejection is a valid HTTP response | `type Rejection: IntoResponse` in both extractor traits |
| State is provided exactly once, before serving | `.with_state(s)` moves the state in; no runtime `None` case |
