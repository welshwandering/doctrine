# Axum Style Guide

> [Doctrine](../../README.md) > [Frameworks](../README.md) > Axum

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119.txt).

Extends [Rust style guide](../languages/rust.md) with Axum-specific conventions.

**Target Version**: Axum 0.8+ with Rust 2024 edition

## Quick Reference

All Rust tooling applies. Additional tools:

| Task | Tool | Command |
|------|------|---------|
| Format | rustfmt | `cargo fmt` |
| Lint | Clippy | `cargo clippy` |
| Test | cargo test | `cargo test` |
| Test API | tower::ServiceExt | See Testing section |
| Database | SQLx | `cargo sqlx prepare` |

## Why Axum?

Axum[^1] is a web framework built on Tower[^2] and Hyper[^3], designed for the Tokio ecosystem with focus on type safety and ergonomics.

**Key advantages**:
- Native async/await with Tokio runtime integration
- Type-safe extractors with compile-time guarantees
- Ergonomic routing with minimal boilerplate
- Composable middleware via Tower
- Zero-cost abstractions with excellent performance[^4]

**When to use Axum**: Choose Axum for greenfield async Rust services requiring type safety, high performance, and composability. Consider Actix-web[^5] for maximum performance or Rocket[^6] for simpler synchronous applications.

## Project Structure

Projects **SHOULD** organize code by feature:

```
my-api/
├── src/
│   ├── main.rs           # Application entry point
│   ├── app.rs            # App factory and router setup
│   ├── config.rs         # Configuration via environment
│   ├── error.rs          # Error types and handlers
│   ├── state.rs          # Shared application state
│   ├── routes/
│   │   ├── mod.rs
│   │   ├── users.rs
│   │   └── health.rs
│   ├── handlers/
│   │   ├── mod.rs
│   │   └── users.rs
│   ├── models/
│   │   ├── mod.rs
│   │   └── user.rs
│   └── middleware/
│       ├── mod.rs
│       └── auth.rs
├── migrations/
├── tests/
├── Cargo.toml
└── .env
```

```rust
// src/main.rs
use my_api::app::create_app;
use my_api::config::Config;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let config = Config::from_env()?;
    let app = create_app(config.clone()).await?;

    let listener = tokio::net::TcpListener::bind(&config.addr).await?;
    axum::serve(listener, app).await?;

    Ok(())
}
```

**Why**: Feature-based organization scales better than layer-based organization. Keeping app creation separate enables testing with different configurations.

## Routing

### Router Composition

Projects **MUST** compose routes using `Router::merge` and `Router::nest`:

```rust
// src/app.rs
use axum::Router;
use crate::routes::{users, health};
use crate::state::AppState;

pub async fn create_app(config: Config) -> anyhow::Result<Router> {
    let state = AppState::new(config).await?;

    let app = Router::new()
        .merge(health::routes())
        .nest("/api/v1", api_routes())
        .with_state(state);

    Ok(app)
}

fn api_routes() -> Router<AppState> {
    Router::new()
        .nest("/users", users::routes())
        .nest("/posts", posts::routes())
}
```

**Why**: Composition via `merge` and `nest` provides better modularity and enables mounting sub-routers at different paths. This pattern allows feature modules to define their own routes independently.

### Method Routing

Routes **SHOULD** use method routing for clarity:

```rust
// src/routes/users.rs
use axum::{Router, routing::{get, post}};
use crate::handlers::users;
use crate::state::AppState;

pub fn routes() -> Router<AppState> {
    Router::new()
        .route("/", get(users::list).post(users::create))
        .route("/:id", get(users::get).put(users::update).delete(users::delete))
}
```

**Do**:
```rust
Router::new()
    .route("/users", get(list_users).post(create_user))
```

**Don't**:
```rust
Router::new()
    .route("/users", get(list_users))
    .route("/users", post(create_user))  // Redundant
```

## Extractors

### Order of Extractors

Extractors **MUST** appear in this order in handler function signatures:

1. `Path` (URL path parameters)
2. `Query` (query string parameters)
3. `State` (application state)
4. `Json` / `Form` (request body)
5. `Extension` (request extensions)
6. `Request` (full request, consumes body)

```rust
use axum::{
    extract::{Path, Query, State},
    Json,
};
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
pub struct Pagination {
    page: Option<u32>,
    per_page: Option<u32>,
}

#[derive(Deserialize)]
pub struct UpdateUserRequest {
    name: Option<String>,
    email: Option<String>,
}

pub async fn update_user(
    Path(user_id): Path<i64>,
    Query(params): Query<Pagination>,
    State(state): State<AppState>,
    Json(payload): Json<UpdateUserRequest>,
) -> Result<Json<User>, AppError> {
    // Handler implementation
}
```

**Why**: This order matches Axum's extractor precedence and makes handlers more readable by placing path parameters first (most specific) and body last (most complex).

### Custom Extractors

Projects **SHOULD** create custom extractors for common patterns:

```rust
// src/extractors/auth.rs
use axum::{
    async_trait,
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
    RequestPartsExt,
};
use axum_extra::headers::{Authorization, authorization::Bearer};
use axum_extra::TypedHeader;

pub struct AuthUser {
    pub user_id: i64,
}

#[async_trait]
impl<S> FromRequestParts<S> for AuthUser
where
    S: Send + Sync,
{
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        let TypedHeader(Authorization(bearer)) = parts
            .extract::<TypedHeader<Authorization<Bearer>>>()
            .await
            .map_err(|_| (StatusCode::UNAUTHORIZED, "Missing authorization header"))?;

        let user_id = verify_token(bearer.token())
            .map_err(|_| (StatusCode::UNAUTHORIZED, "Invalid token"))?;

        Ok(AuthUser { user_id })
    }
}
```

```rust
// Usage in handlers
pub async fn protected_route(
    auth: AuthUser,  // Custom extractor
    State(state): State<AppState>,
) -> Result<Json<User>, AppError> {
    let user = state.db.get_user(auth.user_id).await?;
    Ok(Json(user))
}
```

**Why**: Custom extractors encapsulate common authentication and validation logic, reducing boilerplate and ensuring consistent error handling across routes.

## Error Handling

### Unified Error Type

Projects **MUST** define a unified error type implementing `IntoResponse`:

```rust
// src/error.rs
use axum::{
    response::{IntoResponse, Response},
    http::StatusCode,
    Json,
};
use serde::Serialize;

#[derive(Debug)]
pub enum AppError {
    NotFound,
    Unauthorized,
    Database(sqlx::Error),
    Validation(String),
}

#[derive(Serialize)]
struct ErrorResponse {
    error: String,
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            Self::NotFound => (StatusCode::NOT_FOUND, "Resource not found"),
            Self::Unauthorized => (StatusCode::UNAUTHORIZED, "Unauthorized"),
            Self::Database(_) => (StatusCode::INTERNAL_SERVER_ERROR, "Database error"),
            Self::Validation(msg) => return (
                StatusCode::BAD_REQUEST,
                Json(ErrorResponse { error: msg }),
            ).into_response(),
        };

        (status, Json(ErrorResponse { error: message.to_string() })).into_response()
    }
}

impl From<sqlx::Error> for AppError {
    fn from(err: sqlx::Error) -> Self {
        match err {
            sqlx::Error::RowNotFound => Self::NotFound,
            _ => Self::Database(err),
        }
    }
}
```

**Why**: A unified error type provides consistent API responses and enables using `?` operator in handlers. Implementing `IntoResponse` allows returning errors directly from handlers.

### Using thiserror and anyhow

Projects **SHOULD** use thiserror[^7] for library errors and anyhow[^8] for application errors:

```rust
// src/error.rs
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("Resource not found")]
    NotFound,

    #[error("Unauthorized: {0}")]
    Unauthorized(String),

    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("Validation failed: {0}")]
    Validation(String),
}
```

**Do**:
```rust
use thiserror::Error;  // For typed errors

#[derive(Error, Debug)]
pub enum AppError {
    #[error("User not found")]
    NotFound,
}
```

**Don't**:
```rust
use anyhow::Error;  // Too generic for API errors

pub async fn handler() -> Result<Json<User>, Error> {
    // Don't use anyhow for handler return types
}
```

## Middleware

### Tower Layers

Projects **SHOULD** use Tower layers for cross-cutting concerns:

```rust
use axum::{
    Router,
    middleware::{self, Next},
    http::Request,
    response::Response,
};
use tower_http::{
    trace::TraceLayer,
    cors::CorsLayer,
    compression::CompressionLayer,
};
use std::time::Duration;

pub fn create_app() -> Router {
    Router::new()
        .nest("/api", api_routes())
        .layer(
            ServiceBuilder::new()
                .layer(TraceLayer::new_for_http())
                .layer(CompressionLayer::new())
                .layer(CorsLayer::permissive())
                .layer(middleware::from_fn(timeout_middleware))
        )
}

async fn timeout_middleware<B>(
    req: Request<B>,
    next: Next<B>,
) -> Result<Response, StatusCode> {
    tokio::time::timeout(
        Duration::from_secs(30),
        next.run(req)
    )
    .await
    .map_err(|_| StatusCode::REQUEST_TIMEOUT)
}
```

**Why**: Tower's layer system provides composable, reusable middleware. Using `ServiceBuilder` ensures middleware is applied in the correct order (inner to outer).

### Custom Middleware

Custom middleware **SHOULD** use `middleware::from_fn` for simplicity:

```rust
use axum::{
    middleware::{self, Next},
    http::Request,
    response::Response,
};

async fn auth_middleware<B>(
    req: Request<B>,
    next: Next<B>,
) -> Result<Response, StatusCode> {
    let auth_header = req.headers()
        .get("authorization")
        .and_then(|h| h.to_str().ok());

    match auth_header {
        Some(token) if verify_token(token) => Ok(next.run(req).await),
        _ => Err(StatusCode::UNAUTHORIZED),
    }
}

// Apply to specific routes
let protected = Router::new()
    .route("/admin", get(admin_handler))
    .layer(middleware::from_fn(auth_middleware));
```

**Why**: `from_fn` provides the simplest way to create middleware without implementing `Layer` and `Service` traits manually.

## State Management

### Shared State

Projects **MUST** use `State` extractor for sharing state:

```rust
// src/state.rs
use sqlx::PgPool;

#[derive(Clone)]
pub struct AppState {
    pub db: PgPool,
    pub config: Config,
}

impl AppState {
    pub async fn new(config: Config) -> anyhow::Result<Self> {
        let db = PgPool::connect(&config.database_url).await?;
        Ok(Self { db, config })
    }
}
```

```rust
// src/app.rs
use axum::Router;

pub async fn create_app(config: Config) -> anyhow::Result<Router> {
    let state = AppState::new(config).await?;

    let app = Router::new()
        .route("/users", get(list_users))
        .with_state(state);  // Attach state to router

    Ok(app)
}
```

```rust
// src/handlers/users.rs
pub async fn list_users(
    State(state): State<AppState>,
) -> Result<Json<Vec<User>>, AppError> {
    let users = sqlx::query_as!(User, "SELECT * FROM users")
        .fetch_all(&state.db)
        .await?;

    Ok(Json(users))
}
```

**Why**: `State` is the recommended way to share data across handlers. The state type must implement `Clone`, which is efficient when wrapping `Arc` or connection pools.

### State Guidelines

State types **MUST** implement `Clone` and **SHOULD** use `Arc` for non-Clone fields:

```rust
use std::sync::Arc;

#[derive(Clone)]
pub struct AppState {
    pub db: PgPool,              // Clone implemented (uses Arc internally)
    pub config: Arc<Config>,     // Wrap in Arc for cheap cloning
}
```

**Do**:
```rust
#[derive(Clone)]
pub struct AppState {
    pub pool: PgPool,  // Already uses Arc internally
}
```

**Don't**:
```rust
pub struct AppState {
    pub pool: PgPool,  // Missing Clone derive
}
```

## Testing

### Testing with tower::ServiceExt

Projects **MUST** test handlers using `tower::ServiceExt::oneshot`:

```rust
// tests/users_test.rs
use axum::{
    body::Body,
    http::{Request, StatusCode},
};
use tower::ServiceExt;
use serde_json::json;

#[tokio::test]
async fn test_create_user() {
    let app = create_test_app().await;

    let response = app
        .oneshot(
            Request::builder()
                .method("POST")
                .uri("/api/users")
                .header("content-type", "application/json")
                .body(Body::from(
                    serde_json::to_string(&json!({
                        "name": "Alice",
                        "email": "alice@example.com"
                    })).unwrap()
                ))
                .unwrap()
        )
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::CREATED);

    let body = axum::body::to_bytes(response.into_body(), usize::MAX)
        .await
        .unwrap();
    let user: User = serde_json::from_slice(&body).unwrap();

    assert_eq!(user.name, "Alice");
}
```

**Why**: `tower::ServiceExt` allows testing the router as a `Service` without running an HTTP server, enabling fast, isolated unit tests.

### Test Fixtures

Projects **SHOULD** create test helpers for common patterns:

```rust
// tests/common/mod.rs
use axum::Router;
use sqlx::PgPool;

pub async fn create_test_app() -> Router {
    let config = Config::test();
    let pool = PgPool::connect(&config.database_url).await.unwrap();

    // Run migrations
    sqlx::migrate!().run(&pool).await.unwrap();

    let state = AppState { db: pool, config: Arc::new(config) };
    create_app_with_state(state)
}

pub async fn create_test_user(pool: &PgPool, name: &str) -> User {
    sqlx::query_as!(
        User,
        "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *",
        name,
        format!("{}@example.com", name)
    )
    .fetch_one(pool)
    .await
    .unwrap()
}
```

## Database Integration

### SQLx with Compile-Time Verification

Projects **MUST** use SQLx[^9] with compile-time query verification:

```rust
// src/models/user.rs
use sqlx::FromRow;
use serde::{Deserialize, Serialize};

#[derive(Debug, FromRow, Serialize, Deserialize)]
pub struct User {
    pub id: i64,
    pub name: String,
    pub email: String,
    pub created_at: chrono::DateTime<chrono::Utc>,
}

#[derive(Deserialize)]
pub struct CreateUser {
    pub name: String,
    pub email: String,
}
```

```rust
// src/handlers/users.rs
use sqlx::PgPool;
use axum::{extract::State, Json};

pub async fn create_user(
    State(state): State<AppState>,
    Json(payload): Json<CreateUser>,
) -> Result<Json<User>, AppError> {
    let user = sqlx::query_as!(
        User,
        r#"
        INSERT INTO users (name, email)
        VALUES ($1, $2)
        RETURNING id, name, email, created_at
        "#,
        payload.name,
        payload.email
    )
    .fetch_one(&state.db)
    .await?;

    Ok(Json(user))
}

pub async fn get_user(
    Path(id): Path<i64>,
    State(state): State<AppState>,
) -> Result<Json<User>, AppError> {
    let user = sqlx::query_as!(
        User,
        r#"
        SELECT id, name, email, created_at
        FROM users
        WHERE id = $1
        "#,
        id
    )
    .fetch_one(&state.db)
    .await?;

    Ok(Json(user))
}
```

**Why**: SQLx's `query_as!` macro provides compile-time verification of SQL queries against the database schema, catching errors before runtime. It also generates type-safe bindings automatically.

### Migrations

Projects **MUST** use SQLx migrations:

```bash
# Create migration
sqlx migrate add create_users_table

# Run migrations
sqlx migrate run

# Prepare for offline mode (CI)
cargo sqlx prepare
```

```sql
-- migrations/20240101000000_create_users_table.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
```

**Why**: SQLx migrations integrate with compile-time verification and support offline mode via `cargo sqlx prepare`, enabling CI without a running database.

### Transaction Patterns

Database operations **SHOULD** use transactions for consistency:

```rust
pub async fn transfer_credits(
    from_user_id: i64,
    to_user_id: i64,
    amount: i64,
    pool: &PgPool,
) -> Result<(), AppError> {
    let mut tx = pool.begin().await?;

    sqlx::query!(
        "UPDATE users SET credits = credits - $1 WHERE id = $2",
        amount,
        from_user_id
    )
    .execute(&mut *tx)
    .await?;

    sqlx::query!(
        "UPDATE users SET credits = credits + $1 WHERE id = $2",
        amount,
        to_user_id
    )
    .execute(&mut *tx)
    .await?;

    tx.commit().await?;
    Ok(())
}
```

## See Also

- [Rust Style Guide](../languages/rust.md) - Language-level Rust conventions
- [Testing Guide](../process/testing.md) - General testing principles and patterns
- [CI Guide](../process/ci.md) - Continuous integration best practices

## References

[^1]: [Axum](https://github.com/tokio-rs/axum) - Ergonomic and modular web framework built with Tokio, Tower, and Hyper

[^2]: [Tower](https://github.com/tower-rs/tower) - Modular and reusable components for building robust networking clients and servers

[^3]: [Hyper](https://hyper.rs/) - Fast and safe HTTP implementation for Rust

[^4]: [Axum Performance](https://www.techempower.com/benchmarks/#section=data-r21) - TechEmpower benchmarks showing Axum's performance

[^5]: [Actix-web](https://actix.rs/) - Powerful, pragmatic, and extremely fast web framework for Rust

[^6]: [Rocket](https://rocket.rs/) - Web framework for Rust with focus on ease-of-use, expressiveness, and speed

[^7]: [thiserror](https://github.com/dtolnay/thiserror) - Derive macros for the standard library's `std::error::Error` trait

[^8]: [anyhow](https://github.com/dtolnay/anyhow) - Flexible concrete Error type built on `std::error::Error`

[^9]: [SQLx](https://github.com/launchbadge/sqlx) - Async, pure Rust SQL crate with compile-time checked queries
