+++
title = "Bridging the Gap: A Rustacean Clean Architecture Approach to Web Development"
date = "2024-07-21"
cover = ""
tags = ["Rust", "web", "Clean Architecture", "Axum"]
description = "A clean approach to solve pain points in web development, not just with Rust"
showFullContent = false
+++

## Introduction

Disclaimer: This article is partly made with Claude.

TL;DR: <https://github.com/kigawas/clean-axum>

### The Web Development Framework Dilemma: Navigating Trade-offs in Modern Development

In the dynamic world of web development, choosing the right language and framework often feels like solving a Rubik's cube blindfolded. As developers, we frequently find ourselves balancing competing priorities: performance vs. development speed, flexibility vs. structure, scalability vs. ease of use.

Choosing a web development framework presents a multifaceted challenge. Developers must balance performance, development speed, flexibility, structure, and scalability. This dilemma often leads to compromises, but what if there was a way to optimize for all these factors?

As you'll see later, our proposed solution aims to mitigate these issues while retaining the benefits of a structured approach.

### The full Stack Framework Conundrum: The All-You-Can-Eat Buffet

Full stack frameworks offer a tempting proposition: a complete solution promising rapid development. However, this convenience comes with caveats:

- **Opinionated Architecture**: Great for quick starts, but potentially constraining for custom requirements.
- **Performance Overhead**: Convenience often translates to additional layers, which can impact performance.
- **Steep Learning Curve**: Mastering these frameworks can be time-consuming, potentially offsetting initial productivity gains.

### The Micro Framework Tightrope: The Build-Your-Own Adventure

Micro frameworks offer minimalism and flexibility, but this freedom isn't free:

- **Bare-bones Structure**: Lack of prescribed architecture can lead to inconsistent organization.
- **Architectural Pitfalls**: Without guardrails, it's easy to end up with code that resembles a plate of spaghetti (and not the delicious kind).
- **Integration Overhead**: Manually integrating libraries can be time-consuming and error-prone.

### The Language Dilemma: Static vs. Dynamic

The choice of programming language adds another layer of complexity:

- **Static Languages (e.g., C++, Java, Rust)**:
  - Pros: Strong type safety, excellent performance, and scalability.
  - Cons: Potentially slower development speed, steeper learning curve.

- **Dynamic Languages (e.g., Python, JavaScript)**:
  - Pros: Rapid development, easier to learn and prototype.
  - Cons: Potential performance issues, higher costs at scale, and runtime surprises.

### Seeking the Goldilocks Solution

What if there was a way to get the best of all worlds? A solution that offers:

- The performance and safety of a static language
- The development speed typically associated with dynamic languages
- A structured yet flexible architecture that guides without handcuffing
- Scalability without requiring a second mortgage for your cloud bills

This is where our open-source project comes in.

By leveraging the [Axum](https://github.com/tokio-rs/axum) framework in Rust and implementing clean architecture principles, we've created a scaffold that addresses these common pain points. It offers a balanced approach that we believe can revolutionize how you build web applications.

In the following sections, we'll explore how this scaffold provides a robust foundation for building high-performance, maintainable, and scalable web applications, all while keeping developers happy and productive. No magic wands required – just solid engineering and thoughtful design.

## Clean Architecture in the Context of Axum

In the ever-evolving landscape of web development, architectural patterns play a crucial role in creating maintainable, scalable, and robust applications. [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html), popularized by Robert C. Martin, stands out as a powerful approach to organizing code in a way that maximizes modularity and separation of concerns, especially compared with traditional over-engineered MVC architectures.

> From my personal perspective, MVC architectures may be good for developing user interfaces, but they are not appropriate (or even harmful) for developing web applications intrinsically. The codebase often degenerates into a mess due to so-called "fat models".

### Understanding Clean Architecture

At its core, Clean Architecture is built on a set of principles that promote:

1. **Independence of frameworks**: The architecture doesn't depend on the existence of some library of feature-laden software.
2. **Testability**: Business rules can be tested without the UI, database, web server, or any external element.
3. **Independence of UI**: The UI can change easily, without changing the rest of the system.
4. **Independence of Database**: Business rules are not bound to the database.
5. **Independence of any external agency**: Business rules simply don't know anything about the outside world.

These principles are typically represented in concentric circles, with the innermost circles representing the core business logic, and the outer circles representing interfaces and frameworks.

Clean Architecture in web development emphasizes separation of concerns and modularity. It decouples business logic from frameworks, databases, and UI (in this context, API endpoints). This approach enhances testability, flexibility, and maintainability.

In our implementation, we structure the application in layers: domain models, core business logic, API routers, and external interfaces such as [Utoipa](https://github.com/juhaku/utoipa) OpenAPI documentation (Swagger UI/Scalar) and an optional [shuttle](https://github.com/shuttle-hq/shuttle/) runtime.

### Implementing Clean Architecture

Our project brings these principles to life in the context of Rust and the Axum web framework. Here's how we've structured it to adhere to clean architecture principles:

1. **Domain Models Layer**: At the core, we have our domain models, implemented using [SeaORM](https://github.com/SeaQL/sea-orm). These represent our business entities and are completely independent of any database or framework specifics.

2. **Application Services**: This layer contains our application-specific business rules. In our project, this is represented by the `app::services` module, which handles CRUD operations and other business logic.

3. **Interface Adapters**: This layer adapts data from the format most convenient for use cases and entities, to the format most convenient for some external agency such as a database or the web. In our project, this includes our API routers and dedicated API models such as JSON error responses.

4. **Frameworks and Drivers**: The outermost layer, consisting of frameworks and tools such as Axum, Utoipa and tokio/shuttle runtime. Our project is structured in a way that these can be swapped out with minimal impact on the inner layers.

### Benefits in Rust Web Development

Implementing Clean Architecture in a Rust web application using Axum offers several key benefits:

1. **Modularity**: The clear separation between layers makes it easier to modify or replace components without affecting the entire system.

2. **Testability**: With business logic separated from frameworks, unit testing becomes straightforward and more effective.

3. **Framework Independence**: While we're using Axum, the core business logic is framework-agnostic, allowing for easier transitions if needed.

4. **Database Flexibility**: The separation of domain models from database specifics allows for easier database migrations or even complete database technology changes.

5. **Scalability**: Clean Architecture naturally lends itself to microservices architectures, making it easier to scale specific components of your application independently.

6. **Rust's Safety**: Combining Clean Architecture with Rust's strong type system and ownership model results in exceptionally robust and safe applications.

By adhering to Clean Architecture principles, our project provides a solid foundation for building complex web applications that are easier to maintain, test, and evolve over time. In the following sections, we'll dive deeper into how these principles are implemented in the project structure and key features.

## Project Structure and Key Features

Our project exemplifies clean architecture principles while harnessing the power of Rust and modern web development tools. Let's explore each layer of the architecture and its key features.

### API Logic Layer

The API logic layer, built with Axum, handles HTTP requests and responses with type-safety and ergonomics.

- **Routers and Endpoints**: Utilizes Axum for defining type-safe API routes.
- **Input Validation**: Employs the `Valid` extractor for automatic input validation.
- **Error Handling**: Uses [Anyhow](https://github.com/dtolnay/anyhow)-based custom `ApiError` for consistent error responses.

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

### OpenAPI Documentation

The project leverages Utoipa for comprehensive API documentation:

- Integrates with Utoipa using derive macros like `ToSchema` and `IntoParams`.
- Provides Swagger UI/Scalar for interactive API exploration.

### Database Logic Layer

This layer encapsulates all database-related operations, maintaining separation from the web framework.

- **Services**: Implements CRUD operations and other database manipulations.
- **Framework-Agnostic Design**: Database logic is independent of the web framework.

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

### Domain Models Layer

At the core, we have domain models representing business entities.

- **SeaORM Models**: For defining and working with domain models.
- **Input Parameters, Queries, and Output Schemas**: For input validation, querying, and API responses. These models are ORM library agnostic.

```rust
// Domain Model
#[derive(Clone, Debug, PartialEq, DeriveEntityModel, Eq)]
#[sea_orm(table_name = "user")]
pub struct Model {
    #[sea_orm(primary_key)]
    pub id: i32,
    #[sea_orm(column_type = "Text", unique)]
    pub username: String,
}

// Input Parameter
#[derive(Deserialize, Validate, ToSchema)]
pub struct CreateUserParams {
    #[validate(length(min = 2))]
    pub username: String,
}

// Query
#[derive(Deserialize, Default, IntoParams)]
#[into_params(style = Form, parameter_in = Query)]
pub struct UserQuery {
    #[param(nullable = true)]
    pub username: String,
}

// Output Schema
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

### Key Features

- **Separation of Concerns**: Clear separation between API, database logic, and domain models.
- **Input Validation**: Automatic validation using the `Validate` derive macro and `Valid` extractor.
- **OpenAPI Integration**: Seamless integration with OpenAPI using derive macros.
- **Flexible Database Operations**: Use of SeaORM's `ActiveModel` for flexible database interactions.

## Clean Architecture Implementation Details

Our project embodies clean architecture principles in its very structure. Let's explore how this is achieved.

### Separation of API and DB Logic

The project maintains a clear distinction between API handlers and Application services:

- API handlers (`api::routers`) handle HTTP-related logic and input/output transformations.
- Application services (`app::services`) encapsulate all database operations.

This separation ensures that changes to the API layer don't directly impact the database layer and vice versa.

### Framework and Database Replaceability

The project structure facilitates the replacement of key components:

- The API layer uses Axum, but the core business logic is framework-agnostic.
- While SeaORM is used for database operations, the domain models and business logic are designed to be independent of the specific ORM or database.

### Enforcing Clean Architecture Principles

The project structure naturally enforces clean architecture principles:

- The `models` module contains all domain models, free from external dependencies.
  - Only dependencies for modelling are allowed:
    - `serde` for JSON serialization/deserialization
    - `SeaORM` for domain models
    - `validator` for input validation
    - `utoipa` for OpenAPI documentation.
- The `app` module contains application-specific business rules and interfaces. It depends on `models`.
- The `api` module adapts the core application to the web, handling HTTP concerns. It depends on `app` and `models`.
- The `doc` module contains Utoipa OpenAPI models for generating interactive documentation. It depends on `api` and `models`.

### Extending the Scaffold

When extending the scaffold, developers are guided to maintain architectural boundaries:

1. Add new domain models to the `models` module.
2. Implement business logic in the `app::services` module.
3. Create new API endpoints in the `api::routers` module.
4. Update the `doc` module to include new OpenAPI models.

By following this pattern, new features naturally align with clean architecture principles, ensuring the application remains modular, testable, and maintainable as it grows in complexity.

## Testing Philosophy

We've adopted a comprehensive testing strategy that aligns with our clean architecture principles. Our approach emphasizes the separation of concerns not just in the application code, but also in our test suite. This philosophy ensures that our tests are as modular and maintainable as the code they're verifying.

We maintain a test structure that parallels our main project, facilitating easy navigation and encouraging comprehensive test coverage. Our approach includes isolated test environments, asynchronous testing, and easy CI/CD integration, ensuring robust quality assurance as the project evolves.

### Separation of Test Types

We maintain a clear distinction between two primary types of tests:

1. **API Integration Tests**: These tests verify the behavior of our API endpoints, ensuring that our application correctly handles HTTP requests and responses.

2. **App Service Unit Tests**: These tests focus on the individual components of our business logic, verifying that our services function correctly in isolation.

This separation allows us to pinpoint issues more accurately and maintain a clear understanding of where potential problems may lie.

### Mirroring Project Structure in Tests

One of the key principles of our testing philosophy is to maintain a test folder hierarchy that mirrors our main project structure. Specifically:

- API tests follow the structure of `api::routers`
- App service tests follow the structure of `app::services`

This approach offers several benefits:

- Easy navigation between source code and corresponding tests
- Clear organization of test files
- Encourages developers to write tests alongside new features

Let's look at some examples to illustrate this approach:

#### API Integration Tests

```rust
// tests/api/mod.rs
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

// tests/api/user.rs
pub(super) async fn test_post_users(app: Router) {
    let response = make_post_request(app, "/users", r#"{"username": "test"}"#.to_owned()).await;
    assert_eq!(response.status(), StatusCode::CREATED);

    let body = response.into_body().collect().await.unwrap().to_bytes();
    assert_eq!(&body[..], br#"{"id":1,"username":"test"}"#);
}

// ... other test functions ...
```

#### App Service Unit Tests

```rust
// tests/app/mod.rs
#[tokio::test]
async fn main() -> Result<(), DbErr> {
    let db = setup_test_db("sqlite::memory:").await?;
    test_user(&db).await?;
    Ok(())
}

// tests/app/user.rs
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

### Additional Testing Considerations

Beyond the basic structure, our testing philosophy incorporates several other important aspects:

1. **Isolated Test Environments**: We use in-memory SQLite databases for tests, ensuring that each test runs in a clean, isolated environment.

2. **Comprehensive Coverage**: We aim to test both happy paths and error scenarios, as demonstrated by our `test_post_users_error` function.

3. **Asynchronous Testing**: Our tests are designed to work with Rust's async/await syntax, reflecting the asynchronous nature of our application.

4. **Parameterized Tests**: Where applicable, we use parameterized tests to cover multiple scenarios without duplicating code.

5. **Continuous Integration**: Our test suite can be easily integrated into our CI/CD pipeline, ensuring that all tests pass before merging new code.

By adhering to these principles, we ensure that our test suite remains a valuable tool for maintaining code quality and catching potential issues early in the development process. As the project grows, this testing philosophy will help us maintain confidence in our codebase and facilitate smoother evolution of our application.

## Performance Considerations

When it comes to web development, performance is often a critical factor. Our project doesn't just focus on architectural cleanliness—it's designed with performance in mind.

While specific benchmarks would depend on use cases, our approach is production-ready for high-performance web services without sacrificing code maintainability.

Let's explore some key performance aspects of our approach.

### Rust's Inherent Performance

By choosing Rust, we're already starting on a strong footing. Rust's zero-cost abstractions, compile-time checks, and lack of a garbage collector contribute to excellent runtime performance. This is particularly beneficial for web services that need to handle high concurrency and low latency.

### Axum's Efficiency

Axum, built on top of hyper and tokio, is designed for high performance. Its tower-based middleware approach and efficient routing contribute to minimal overhead. Our project leverages these strengths, ensuring that the architectural benefits don't come at the cost of speed.

### Database Optimization

While our project uses SeaORM for database operations, the clean architecture allows for easy optimization:

1. **Query Optimization**: The separation of database logic in the `app::services` module allows for fine-tuning of database queries without affecting the rest of the application.

2. **Connection Pooling Configuration**: We can adjust connection pooling parameters to maximize performance on our specific dedicated servers, whether on-premises or in the cloud.

3. **Baked Queries**: We can use baked queries to reduce the overhead of converting ORM queries to SQL statements.

### Scalability Through Clean Architecture

The clean architecture of our project contributes to its scalability:

1. **Separation of Concerns**: Makes it easier to optimize or replace individual components without affecting the entire system.

2. **Stateless Design**: Our API handlers are designed to be stateless, facilitating horizontal scaling.

3. **Easy Caching Integration**: The clear separation of layers makes it straightforward to introduce caching at various levels of the application.

While performance optimizations can always be made, our project provides a sweet spot that doesn't sacrifice speed for cleanliness. As always, we recommend profiling and benchmarking for specific use cases to identify any bottlenecks and optimize accordingly.

## Conclusion

As we wrap up our journey through our project, let's reflect on how it addresses the web development framework dilemma we started with.

Remember the tug-of-war between full-stack frameworks and micro-frameworks? The struggle to balance development speed, performance, and maintainability? Our project offers a compelling solution to these challenges.

By leveraging Rust's performance and safety, Axum's efficiency, and the principles of clean architecture, we've created a scaffold that:

1. **Provides Structure Without Rigidity**: Unlike opinionated full-stack frameworks, our project offers a clear structure that guides development without constraining creativity.

2. **Ensures High Performance**: We harness Rust and Axum's speed while our architectural choices facilitate optimizations and scalability.

3. **Maintains Flexibility**: The clear separation of concerns allows for easy swapping of components, be it the database, the web framework, or even moving to a different architecture. If you don't like any part of our project, you can always modify it effortlessly.

4. **Enhances Maintainability**: With a clear project structure and comprehensive testing philosophy, we've set the stage for long-term maintainability.

5. **Speeds Up Development**: While not as instant as some dynamically-typed frameworks, our approach is still simple enough to accelerate development as projects grow in complexity.

This approach demonstrates that with careful design, we can achieve performance, safety, and clean code simultaneously in web development. We believe our Clean Architecture approach with Rust and Axum offers a compelling solution to common web development challenges.

You are invited to explore, build with, and contribute to this evolving project on [GitHub](https://github.com/kigawas/clean-axum).
