
+++
title = "A Rustacean Clean Architecture Approach to Web Development"
date = "2024-07-21"
cover = ""
tags = ["Rust", "web", "Clean Architecture", "Axum"]
description = "A clean approach to solve pain points in web development, not just with Rust"
showFullContent = false
+++

## Introduction

TL;DR: <https://github.com/kigawas/clean-axum>

### The Problem with MVC

Most web frameworks — Django, Rails, Laravel — follow MVC patterns that degenerate over time. Business logic creeps into models, controllers balloon with validation, and the codebase becomes a tangle where everything depends on everything.

Full-stack frameworks offer rapid starts but impose opinionated structures that constrain you later. Micro-frameworks offer freedom but no guardrails, leading to inconsistent architecture. And the language choice adds tension: static languages give you safety and performance at the cost of velocity; dynamic languages give you speed at the cost of runtime surprises.

What if you could get structure without rigidity, performance without sacrificing development speed?

That's what [clean-axum](https://github.com/kigawas/clean-axum) aims to provide: a Rust scaffold built on [Axum](https://github.com/tokio-rs/axum) that applies Clean Architecture principles to web development.

## Clean Architecture in Axum

[Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html), popularized by Robert C. Martin, organizes code around one fundamental rule:

> **Dependencies flow inward.** Outer layers depend on inner layers. Inner layers know nothing about the outside world.

In practice, this means:

- **Domain models** define data structures with no knowledge of databases, frameworks, or HTTP.
- **Persistence** handles all business logic and database operations, depending only on models.
- **API routers** translate HTTP into persistence calls, depending on both layers above.

This naturally produces independence from frameworks, databases, and external interfaces — making each layer testable and replaceable in isolation.

> From my personal perspective, while MVC architectures may be suitable for developing user interfaces, they can be intrinsically detrimental when applied to web applications. The codebase often degenerates into a mess due to so-called "fat models".

### How the Layers Map to Code

```text
┌─────────────────────────────────────────────┐
│  api/routers/       HTTP plumbing (thin)     │
├─────────────────────────────────────────────┤
│  app/persistence/   Business logic (thick)   │
├─────────────────────────────────────────────┤
│  models/            Data definitions (slim)  │
│    domains/  params/  schemas/  queries/     │
└─────────────────────────────────────────────┘
```

- `models/` depends on nothing. It contains domain entities, input parameters, output schemas, and query filters.
- `app/` depends on `models/`. All CRUD operations and business rules live in `app::persistence`.
- `api/` depends on `app/` and `models/`. Routers are thin — they parse HTTP and delegate to persistence.
- `doc/` depends on `api/` and `models/`. It contains [Utoipa](https://github.com/juhaku/utoipa) OpenAPI models for Swagger UI/Scalar.

## Project Structure and Key Features

### Domain Models Layer

At the core, domain models represent business entities. Input parameters, queries, and output schemas are separate structs — all ORM-agnostic except for the entity definitions themselves.

```rust
// Domain Model (ORM-coupled — this is the only place)
#[derive(Clone, Debug, PartialEq, DeriveEntityModel, Eq)]
#[sea_orm(table_name = "user")]
pub struct Model {
    #[sea_orm(primary_key)]
    pub id: i32,
    #[sea_orm(column_type = "Text", unique)]
    pub username: String,
}

// Input Parameter (ORM-agnostic)
#[derive(Deserialize, Validate, ToSchema)]
pub struct CreateUserParams {
    #[validate(length(min = 2))]
    pub username: String,
}

// Query (ORM-agnostic)
#[derive(Deserialize, Default, IntoParams)]
#[into_params(style = Form, parameter_in = Query)]
pub struct UserQuery {
    #[param(nullable = true)]
    pub username: String,
}

// Output Schema (ORM-agnostic)
#[derive(Serialize, ToSchema)]
pub struct UserSchema {
    pub id: u32,
    pub username: String,
}

impl From<user::Model> for UserSchema {
    fn from(user: user::Model) -> Self {
        Self {
            id: user.id as u32,
            username: user.username,
        }
    }
}
```

Only the domain model struct uses [SeaORM](https://github.com/SeaQL/sea-orm) derives. Everything else — params, queries, schemas — is plain Rust with `serde`, `validator`, and `utoipa`. This containment is deliberate: ORM coupling exists, but it's confined to one place.

### Persistence Layer

This is where you'll spend most of your time. All database operations and business rules live here, separated from HTTP concerns.

```rust
pub async fn create_user(
    db: &DbConn,
    params: CreateUserParams,
) -> Result<user::ActiveModel, DbErr> {
    user::ActiveModel {
        username: Set(params.username),
        ..Default::default()
    }
    .save(db)
    .await
}

pub async fn search_users(db: &DbConn, query: UserQuery) -> Result<Vec<user::Model>, DbErr> {
    user::Entity::find()
        .filter(user::Column::Username.contains(query.username))
        .all(db)
        .await
}
```

Because persistence functions take a `DbConn` and return domain types, they're straightforward to test with an in-memory database — no HTTP server needed.

### API Logic Layer

Routers translate HTTP into persistence calls. They should be boring — if a router function exceeds a few lines, logic is leaking into the wrong layer.

```rust
async fn users_post(
    state: State<AppState>,
    Valid(Json(params)): Valid<Json<CreateUserParams>>,
) -> Result<impl IntoResponse, ApiError> {
    let user = create_user(&state.conn, params).await
        .map_err(ApiError::from)?;

    let user = user.try_into_model().unwrap();
    Ok((StatusCode::CREATED, Json(UserSchema::from(user))))
}

async fn users_get(
    state: State<AppState>,
    query: Option<Query<UserQuery>>,
) -> Result<impl IntoResponse, ApiError> {
    let Query(query) = query.unwrap_or_default();

    let users = search_users(&state.conn, query).await
        .map_err(ApiError::from)?;
    Ok(Json(UserListSchema::from(users)))
}
```

Key features at this layer:

- **Input Validation**: The `Valid` extractor triggers automatic validation via `validator` derives.
- **Error Handling**: [Anyhow](https://github.com/dtolnay/anyhow)-based `ApiError` provides consistent JSON error responses.
- **OpenAPI Integration**: Utoipa derive macros generate interactive Swagger UI/Scalar documentation.

### What "Replaceable" Actually Means

The database **engine** is replaceable — SQLite for tests, Postgres for production, no code changes. The ORM is not — `models/domains/` uses SeaORM derives, and `app::persistence/` uses SeaORM's query API. But that's the right trade-off: you get type-safe queries and migrations in exchange for coupling to one ORM. The coupling is contained to two directories; everything else is plain Rust structs.

The web framework is similarly contained. Swap Axum for Actix — you rewrite `api/routers/` and nothing else changes. The persistence layer doesn't know HTTP exists.

## Testing Philosophy

### Test Persistence, Not Routers

Your routers are short functions that delegate to persistence. If persistence works, the router works. Put your coverage effort into `app::persistence`, and write one integration test per endpoint to verify wiring.

The test folder hierarchy mirrors the main project structure:

- `tests/app/` mirrors `app::persistence`
- `tests/api/` mirrors `api::routers`

#### Persistence Unit Tests

```rust
#[tokio::test]
async fn user_main() -> Result<(), DbErr> {
    let db = setup_test_db("sqlite::memory:").await?;
    test_user(&db).await?;
    Ok(())
}

pub(super) async fn test_user(db: &DatabaseConnection) -> Result<(), DbErr> {
    let params = CreateUserParams {
        username: "test".to_string(),
    };

    let user = create_user(db, params).await?;
    let expected = user::ActiveModel {
        id: Unchanged(1),
        username: Unchanged("test".to_owned()),
    };
    assert_eq!(user, expected);
    Ok(())
}
```

#### API Integration Tests

```rust
#[tokio::test]
async fn user_main() {
    let db = setup_test_db("sqlite::user?mode=memory&cache=shared")
        .await
        .expect("Set up db failed!");

    let app = setup_router(db);
    test_post_users(app.clone()).await;
    test_post_users_error(app.clone()).await;
    test_get_users(app).await;
}

pub(super) async fn test_post_users(app: Router) {
    let response = make_post_request(app, "/users", r#"{"username": "test"}"#.to_owned()).await;
    assert_eq!(response.status(), StatusCode::CREATED);

    let body = response.into_body().collect().await.unwrap().to_bytes();
    assert_eq!(&body[..], br#"{"id":1,"username":"test"}"#);
}
```

Tests use in-memory SQLite databases for isolation. Both happy paths and error scenarios are covered (e.g., `test_post_users_error`). All tests are async, reflecting the application's async nature.

## Adding a Feature

Every feature follows the same four steps:

1. **`models/`** — Define your domain entity, input params, output schema, and query filter.
2. **`app/persistence/`** — Implement the CRUD functions and business logic.
3. **Write tests** — Cover the persistence layer thoroughly; add an integration test for wiring.
4. **`api/routers/`** — Wire it to HTTP. Update `doc/` for OpenAPI.

Step 4 should take minutes. If it takes longer, logic is leaking into the wrong layer.

## Conclusion

Clean Architecture isn't about following a concentric circles diagram. It's about one rule: **dependencies flow inward**. Structure your code so that models know nothing about persistence, persistence knows nothing about HTTP, and the result is a codebase where every layer can be tested, replaced, and reasoned about independently.

By combining this principle with Rust's type safety and Axum's efficiency, [clean-axum](https://github.com/kigawas/clean-axum) provides a scaffold that is structured without being rigid, performant without sacrificing clarity, and maintainable as complexity grows.

Explore, build with, and contribute to the project on [GitHub](https://github.com/kigawas/clean-axum).
