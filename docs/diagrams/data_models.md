# axum — Core Type / Trait ER Diagram

axum is a framework, not a data-persistence layer, so it has no database schema.
This diagram maps the key **types and traits** and how they relate — the closest
equivalent of a data model in a framework codebase.

```mermaid
erDiagram

    ROUTER {
        PathRouter  path_router
        State       state
        Vec         fallbacks
        Vec         layers
    }

    PATH_ROUTER {
        matchit_Router  inner
        HashMap         route_id_to_endpoint
        u32             next_route_id
    }

    METHOD_ROUTER {
        MethodFilter    allowed_methods
        Route           get_handler
        Route           post_handler
        Route           put_handler
        Route           delete_handler
        Route           patch_handler
        Route           head_handler
        Route           options_handler
        Route           trace_handler
        Option          fallback
    }

    ROUTE {
        BoxedHandler    service
    }

    REQUEST {
        Method      method
        Uri         uri
        Version     version
        HeaderMap   headers
        Extensions  extensions
        Body        body
    }

    RESPONSE {
        StatusCode  status
        HeaderMap   headers
        Extensions  extensions
        Body        body
    }

    BODY {
        BoxBody     inner
    }

    HANDLER_TRAIT {
        string  call_signature
    }

    FROM_REQUEST_TRAIT {
        string  type_alias  "type Rejection: IntoResponse"
    }

    FROM_REQUEST_PARTS_TRAIT {
        string  type_alias  "type Rejection: IntoResponse"
    }

    INTO_RESPONSE_TRAIT {
        string  method  "fn into_response(self) -> Response"
    }

    EXTRACTOR_PATH {
        T   inner   "serde::Deserialize"
    }

    EXTRACTOR_QUERY {
        T   inner   "serde::Deserialize"
    }

    EXTRACTOR_JSON {
        T   inner   "serde::Deserialize"
    }

    EXTRACTOR_FORM {
        T   inner   "serde::Deserialize"
    }

    EXTRACTOR_STATE {
        T   inner   "Clone + Send + Sync"
    }

    EXTRACTOR_EXTENSION {
        T   inner   "Clone + Send + Sync"
    }

    TOWER_SERVICE_TRAIT {
        string  associated_type  "Request, Response, Error, Future"
    }

    TOWER_LAYER_TRAIT {
        string  method  "fn layer(self, inner: S) -> Self::Service"
    }

    %% Structural relationships
    ROUTER             ||--o{ PATH_ROUTER      : "contains"
    ROUTER             ||--o{ TOWER_LAYER_TRAIT : "stacks"
    PATH_ROUTER        ||--o{ METHOD_ROUTER    : "maps route_id →"
    METHOD_ROUTER      ||--o{ ROUTE            : "holds per-method"
    ROUTE              ||--|| TOWER_SERVICE_TRAIT : "wraps"

    %% Request / Response flow
    REQUEST            ||--|| BODY             : "carries"
    RESPONSE           ||--|| BODY             : "carries"

    %% Handler trait relationships
    HANDLER_TRAIT      ||--|| TOWER_SERVICE_TRAIT   : "converts to"
    HANDLER_TRAIT      ||--o{ FROM_REQUEST_PARTS_TRAIT : "auto-extracts args via"
    HANDLER_TRAIT      ||--o{ FROM_REQUEST_TRAIT       : "auto-extracts body via"
    HANDLER_TRAIT      ||--|| INTO_RESPONSE_TRAIT      : "return type must impl"

    %% Concrete extractor implementations
    EXTRACTOR_PATH      ||--|| FROM_REQUEST_PARTS_TRAIT : "implements"
    EXTRACTOR_QUERY     ||--|| FROM_REQUEST_PARTS_TRAIT : "implements"
    EXTRACTOR_STATE     ||--|| FROM_REQUEST_PARTS_TRAIT : "implements"
    EXTRACTOR_EXTENSION ||--|| FROM_REQUEST_PARTS_TRAIT : "implements"
    EXTRACTOR_JSON      ||--|| FROM_REQUEST_TRAIT       : "implements (consumes body)"
    EXTRACTOR_FORM      ||--|| FROM_REQUEST_TRAIT       : "implements (consumes body)"

    %% IntoResponse implementations
    RESPONSE            ||--|| INTO_RESPONSE_TRAIT : "is the canonical impl"
    EXTRACTOR_JSON      ||--|| INTO_RESPONSE_TRAIT : "also implements (as response)"

    %% Middleware
    TOWER_LAYER_TRAIT   ||--|| TOWER_SERVICE_TRAIT : "wraps"
```
