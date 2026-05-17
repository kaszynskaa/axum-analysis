# axum — Sequence Diagram: HTTP Request Lifecycle

Full message sequence for a `POST /users` request handled by
`create_user(State<AppState>, Json<CreateUser>) -> Json<User>`.

Each actor corresponds to a real struct or trait in the axum source tree.

```mermaid
sequenceDiagram
    autonumber
    participant C  as Client
    participant L  as TcpListener<br/>(Tokio)
    participant SV as axum::serve<br/>serve/mod.rs
    participant HY as hyper-util<br/>(external)
    participant R  as Router<br/>routing/mod.rs
    participant PR as PathRouter<br/>path_router.rs
    participant MR as MethodRouter<br/>method_routing.rs
    participant HN as Handler::call<br/>handler/mod.rs
    participant EP as FromRequestParts<br/>extractors
    participant EB as FromRequest<br/>extractor (body)
    participant U  as user async fn

    C  ->>  L:  TCP SYN
    L  ->>  SV: accept() yields (TcpStream, SocketAddr)

    Note over SV: handle_connection() L559<br/>clones Router via IntoMakeService<br/>spawns tokio task

    SV ->>  HY: serve_connection_with_upgrades(TowerToHyperService)
    HY ->>  HY: parse raw bytes → Request<Incoming>
    HY ->>  R:  Service::call(req)

    Note over R: Router::call() L614<br/>wraps body → axum::Body<br/>calls call_with_state(req, ())

    R  ->>  PR: call_with_state(req, state)

    Note over PR: inserts OriginalUri into extensions<br/>(feature = original-uri)

    PR ->>  PR: matchit node.at(uri.path()) L342

    alt path matched → RouteId found
        PR ->>  PR: url_params::insert_url_params() L353<br/>stores captures into extensions
        PR ->>  PR: set_matched_path_for_request() L346<br/>stores route template into extensions

        PR ->>  MR: call_with_state(req, state)

        MR ->>  MR: call! macro L1193<br/>if req.method() == POST

        alt method handler registered
            MR ->>  MR: BoxedHandler.into_route(state)<br/>bakes state into Route
            MR ->>  HN: route.oneshot_inner_owned(req)

            Note over HN: req.into_parts() L239<br/>splits (parts, body)

            loop each FromRequestParts arg (State, Path, Query…)
                HN ->>  EP: from_request_parts(&mut parts, &state)
                EP -->> HN: Ok(value)
            end

            HN ->>  EB: from_request(Request::from_parts(parts,body), &state)

            Note over EB: Json: checks Content-Type L107<br/>buffers body → Bytes L111<br/>serde_json::from_slice L184

            EB -->> HN: Ok(Json(payload))
            HN ->>  U:  handler(State(app_state), Json(payload))
            U  -->> HN: Json(user)  [impl IntoResponse]
            HN ->>  HN: Json::into_response() L201<br/>serde_json::to_writer L229<br/>→ 200 OK + Content-Type: application/json

        else method not registered → 405
            MR -->> R:  405 Method Not Allowed<br/>+ Allow: GET, POST, … header
        end

    else no path match → 404
        PR -->> R:  Err((req, state))
        R  ->>  R:  catch_all_fallback.call_with_state(req, ())<br/>→ 404 Not Found
    end

    alt any FromRequestParts extraction failed
        EP -->> HN: Err(rejection)
        HN ->>  HN: rejection.into_response()
    end

    alt FromRequest extraction failed
        EB -->> HN: Err(JsonRejection)
        HN ->>  HN: JsonRejection::into_response()<br/>415 / 422 / 400
    end

    R  -->> HY: Response (200 / 404 / 405 / 4xx / 5xx)
    HY ->>  HY: frame as HTTP/1.1 or HTTP/2 bytes
    HY -->> C:  HTTP response
```

## Actor map

| Actor | File | Line |
|-------|------|------|
| `axum::serve` | `axum/src/serve/mod.rs` | L103, L559 |
| `Router` | `axum/src/routing/mod.rs` | L609, L452 |
| `PathRouter` | `axum/src/routing/path_router.rs` | L325 |
| `MethodRouter` | `axum/src/routing/method_routing.rs` | L1167 |
| `Handler::call` | `axum/src/handler/mod.rs` | L238 |
| `FromRequestParts` | `axum-core/src/extract/mod.rs` | — |
| `FromRequest` (Json) | `axum/src/json.rs` | L99 |
| `IntoResponse` (Json) | `axum/src/json.rs` | L197 |
