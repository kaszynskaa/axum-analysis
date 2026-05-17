# axum Analysis Project — Mermaid Diagrams

Three diagrams covering angles not yet in `docs/diagrams/`: workspace code
organisation, route registration build-time flow, and the project's own
document structure.

---

## 1. Architecture — Workspace & Crate Dependency Map

How the four published crates are organised, what each exports, and which
external crates they pull in.

```mermaid
flowchart TB
    subgraph WS["axum Cargo Workspace — Rust Edition 2021, MSRV 1.80"]
        direction TB

        subgraph AM["axum-macros v0.5.1"]
            DH["#[debug_handler]"]
            DFRQ["#[derive(FromRequest)]\n#[derive(FromRequestParts)]"]
            DTP["#[derive(TypedPath)]"]
        end

        subgraph AC["axum-core v0.5.6 — stable base for library authors"]
            FR["trait FromRequest&lt;S&gt;\nconsumes body · last arg only"]
            FRP["trait FromRequestParts&lt;S&gt;\nborrow-only · non-body extractors"]
            IR["trait IntoResponse\nall handler return types"]
            BD["struct Body\nnewtype over BoxBody"]
        end

        subgraph AX["axum v0.8.9 — application framework"]
            direction LR
            subgraph RO["routing/"]
                RTR["Router&lt;S&gt;\nroute nest merge layer\nwith_state fallback"]
                PTR["PathRouter\nmatchit radix tree\nUrlParams MatchedPath"]
                MTR["MethodRouter\nget post put delete…\nAllow header · 405"]
            end
            subgraph EX["extract/"]
                EPath["Path&lt;T&gt;  State&lt;T&gt;"]
                EQuery["Query&lt;T&gt;  Extension&lt;T&gt;"]
                EJson["Json&lt;T&gt;  Form&lt;T&gt;"]
                EWS["Multipart  WebSocketUpgrade"]
            end
            subgraph HA["handler/"]
                HT["trait Handler&lt;T,S&gt;\nimpl_handler! macro\narity 0–16"]
            end
            subgraph SV["serve/"]
                SF["serve(listener, router)\naccept loop · graceful shutdown"]
            end
            subgraph RE["response/"]
                RR["Html&lt;T&gt;  Redirect\nSse&lt;S&gt;  NoContent"]
            end
        end

        subgraph AE["axum-extra v0.12.6 — optional feature-gated add-ons"]
            direction LR
            ETP["TypedPath\ncompile-time route safety"]
            ETH["TypedHeader  CookieJar"]
            EPB["Protobuf  JsonLines"]
        end
    end

    subgraph ExtA["proc-macro only"]
        SYN["syn  quote  proc-macro2"]
    end

    subgraph ExtB["HTTP primitives"]
        HTTP["http 1.0  http-body 1.0\nhttp-body-util 0.1  bytes 1.7"]
        TWR["tower 0.5  tower-service 0.3\ntower-layer 0.3  tower-http 0.6"]
    end

    subgraph ExtC["Runtime & routing"]
        TOK["tokio 1.44"]
        HYP["hyper 1.4  hyper-util 0.1\nHTTP/1.1 + HTTP/2"]
        MIT["matchit 0.9 radix tree"]
        SRD["serde 1.0  serde_json 1.0\nmime  tracing  multer"]
    end

    AM  -->|"no axum dep"| SYN
    AC  --> HTTP
    AC  --> TWR
    AX  -->|"depends on"| AC
    AX  --> TOK
    AX  --> HYP
    AX  --> MIT
    AX  --> SRD
    AE  -->|"depends on"| AX
    AX  -.->|"ergonomics via (optional)"| AM
```

---

## 2. Sequence — Route Registration (Build Time)

How `.route()`, `.layer()`, and `.with_state()` build the internal data
structures **before** the first request arrives. Complements the existing
request-handling sequence diagrams.

```mermaid
sequenceDiagram
    autonumber
    participant App  as Application code
    participant R    as Router&lt;S&gt;
    participant RI   as RouterInner&lt;S&gt;
    participant PR   as PathRouter&lt;S&gt;
    participant Node as Node (matchit)
    participant MR   as MethodRouter&lt;S&gt;

    App  ->>  R:    Router::new()
    R    ->>  RI:   RouterInner { path_router: default, default_fallback: true }
    RI   ->>  PR:   PathRouter { routes: [], node: empty }
    R    -->> App:  Router&lt;S&gt; (empty)

    App  ->>  R:    .route("/users", get(handler))
    R    ->>  RI:   into_inner() — Arc::try_unwrap or clone
    RI   ->>  PR:   route("/users", MethodRouter)
    PR   ->>  PR:   validate_path("/users")
    PR   ->>  Node: insert("/users", RouteId(0))
    Note over Node: matchit radix tree stores\n"/users" → RouteId(0)
    PR   ->>  PR:   routes.push(Endpoint::MethodRouter(mr))
    Note over MR:   MethodEndpoint::BoxedHandler stored\n(state S not yet known)
    R    -->> App:  Router&lt;S&gt; with one route

    App  ->>  R:    .route("/users/{id}", get(get_user))
    R    ->>  PR:   route("/users/{id}", MethodRouter)
    PR   ->>  Node: insert("/users/{id}", RouteId(1))
    PR   ->>  PR:   routes.push(Endpoint::MethodRouter(mr2))
    R    -->> App:  Router&lt;S&gt; with two routes

    App  ->>  R:    .layer(TraceLayer)
    R    ->>  PR:   layer(TraceLayer)
    PR   ->>  PR:   map each Endpoint through layer.layer(route)
    Note over PR:   Every Route is now wrapped:\nTraceLayer(Route(handler))
    R    -->> App:  Router&lt;S&gt; with layered routes

    App  ->>  R:    .with_state(app_state)
    R    ->>  RI:   PathRouter::with_state(state)
    loop  each Endpoint::MethodRouter in routes
        RI   ->>  MR:   with_state(state.clone())
        MR   ->>  MR:   BoxedHandler.into_route(state)\nBoxedHandler → Route
        Note over MR:   State baked in. MethodEndpoint::BoxedHandler\nbecomes MethodEndpoint::Route
    end
    R    -->> App:  Router&lt;()&gt; — ready for axum::serve()

    App  ->>  R:    axum::serve(listener, router)
    Note over R:    IntoMakeService clones Router&lt;()&gt;\nonce per accepted TCP connection
```

---

## 3. ER — Project Document Structure

Entity-relationship map of the analysis project's own files: how document
types relate to each other and to the external axum source they describe.

```mermaid
erDiagram

    AXUM_SOURCE {
        string  repo    "github.com/tokio-rs/axum"
        string  crate   "axum / axum-core / axum-extra / axum-macros"
        string  file    "e.g. routing/mod.rs"
        int     line    "line number reference"
    }

    REFERENCE_DOC {
        string  path    "docs/*.md"
        string  kind    "architecture | entry-points | quality-report | request-trace | top-level-dirs"
        string  title
    }

    DIAGRAM {
        string  path       "docs/diagrams/*.md"
        string  kind       "flowchart | sequenceDiagram | erDiagram | stateDiagram | classDiagram"
        string  title
        boolean has_html_preview
    }

    ANSWER {
        string  path       "answers/*.md"
        string  prompt     "original question / task"
        boolean superseded
    }

    IMPROVED_ANSWER {
        string  path    "answers/improved answers/*.md"
        string  prompt
    }

    SUBPART_ANSWER {
        string  path    "answers/answer for last subpart/*.md"
        int     order   "01_ 02_ prefix"
    }

    CLAUDE_MD {
        string  path    "CLAUDE.md"
        string  section "commands | layout | diagram-inventory | architecture | refactoring-notes"
    }

    AXUM_SOURCE        ||--o{  REFERENCE_DOC   : "analysed in"
    AXUM_SOURCE        ||--o{  DIAGRAM         : "visualised in"
    AXUM_SOURCE        ||--o{  ANSWER          : "described in"

    REFERENCE_DOC      ||--o{  DIAGRAM         : "expanded into"
    REFERENCE_DOC      }o--||  CLAUDE_MD       : "summarised in"

    ANSWER             ||--o|  IMPROVED_ANSWER : "revised as"
    ANSWER             ||--o{  SUBPART_ANSWER  : "split into"

    DIAGRAM            }o--||  CLAUDE_MD       : "inventoried in"
    REFERENCE_DOC      }o--||  CLAUDE_MD       : "inventoried in"
```
