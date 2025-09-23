# Architecture Overview

This document provides a comprehensive overview of the URL Shortener application architecture, designed to help contributors understand the system structure and design decisions.

## 🏗️ High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              HTTP Layer                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌───────────┐ │
│  │    Static    │    │   Admin UI   │    │     API      │    │  Health   │ │
│  │    Files     │    │   /admin     │    │  Endpoints   │    │   Check   │ │
│  │              │    │              │    │              │    │           │ │
│  └──────────────┘    └──────────────┘    └──────────────┘    └───────────┘ │
│                                                  │                          │
├─────────────────────────────────────────────────┼──────────────────────────┤
│                         Middleware Layer        │                          │
│                                                  │                          │
│                         ┌────────────────────────▼─────────────────────────┐│
│                         │         API Key Authentication                   ││
│                         │         (Protected Routes Only)                  ││
│                         └────────────────────────┬─────────────────────────┘│
├──────────────────────────────────────────────────┼──────────────────────────┤
│                       Application Layer          │                          │
│                                                   │                          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────▼─────────────┐            │
│  │   URL Redirect  │  │  URL Shortening │  │   Template        │            │
│  │    Handler      │  │     Handler     │  │   Rendering       │            │
│  │                 │  │                 │  │                   │            │
│  └─────────────────┘  └─────────────────┘  └───────────────────┘            │
│           │                     │                     │                     │
├───────────┼─────────────────────┼─────────────────────┼─────────────────────┤
│           │                     │                     │                     │
│           ▼                     ▼                     ▼                     │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                     Application State                                │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────────────┐│  │
│  │  │  Database   │  │   API Key   │  │       Template Directory        ││  │
│  │  │   Handle    │  │             │  │                                 ││  │
│  │  └─────────────┘  └─────────────┘  └─────────────────────────────────┘│  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│           │                                                                 │
├───────────┼─────────────────────────────────────────────────────────────────┤
│           ▼                                                                 │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                      Database Layer                                  │  │
│  │                                                                      │  │
│  │  ┌─────────────────┐              ┌─────────────────────────────────┐ │  │
│  │  │  UrlDatabase    │◄─────────────┤      SQLite Implementation     │ │  │
│  │  │     Trait       │              │                                 │ │  │
│  │  │                 │              │  ┌─────────────────────────────┐│ │  │
│  │  │ - insert_url()  │              │  │     Connection Pool        ││ │  │
│  │  │ - get_url()     │              │  │                             ││ │  │
│  │  └─────────────────┘              │  └─────────────────────────────┘│ │  │
│  │                                   └─────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                           │                                 │
└───────────────────────────────────────────┼─────────────────────────────────┘
                                            ▼
                                   ┌─────────────────┐
                                   │  SQLite File    │
                                   │   database.db   │
                                   └─────────────────┘
```

## 🔧 Core Components

### 1. Application Entry Point (`src/bin/main.rs`)

The main entry point orchestrates the application startup:

- **Telemetry Initialization**: Sets up structured logging with tracing
- **Configuration Loading**: Reads YAML configs and environment variables
- **Application Bootstrap**: Creates and starts the web server
- **Graceful Shutdown**: Handles SIGTERM/SIGINT signals

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // Initialize logging → Load config → Build app → Run until stopped
}
```

### 2. Configuration System (`src/lib/configuration.rs`)

Multi-layered configuration management using Figment:

- **Base Configuration**: `configuration/base.yml`
- **Environment Overrides**: `configuration/local.yml` or `configuration/production.yml`
- **Environment Variables**: `APP_*` prefixed variables
- **Type Safety**: Strongly typed configuration structs with validation

**Configuration Hierarchy**: Environment Variables > Environment File > Base File

### 3. Database Layer (`src/lib/database/`)

**Design Pattern**: Repository Pattern with Trait Abstraction

```rust
#[async_trait]
pub trait UrlDatabase: Send + Sync {
    async fn insert_url(&self, id: &str, url: &str) -> Result<(), DatabaseError>;
    async fn get_url(&self, id: &str) -> Result<String, DatabaseError>;
}
```

**Benefits**:
- **Testability**: Easy to mock for unit tests
- **Flexibility**: Can swap database implementations (PostgreSQL, MySQL, etc.)
- **Type Safety**: SQLx provides compile-time SQL validation

**Current Implementation**: SQLite with connection pooling and automatic migrations

### 4. HTTP Layer (`src/lib/routes/`)

RESTful API design with clear separation of concerns:

#### Endpoints:

- **`POST /api/shorten`** (Protected) - Create shortened URLs
- **`GET /api/redirect/{id}`** - Redirect to original URL
- **`GET /api/health_check`** - Service health status
- **`GET /admin`** - Admin web interface
- **Static Files** - CSS/JS assets

#### Route Handlers:
Each handler follows a consistent pattern:
1. Extract request data (path params, body, headers)
2. Validate input
3. Interact with database layer
4. Return structured response or error

### 5. Middleware Stack (`src/lib/middleware.rs`)

**Request Processing Pipeline**:
```
Request → Request ID → Tracing → API Key Auth → Handler → Response
```

- **Request ID Generation**: UUID-based correlation IDs
- **Distributed Tracing**: Request lifecycle tracking
- **API Key Authentication**: Protects sensitive endpoints
- **Error Handling**: Converts errors to HTTP responses

### 6. Application State (`src/lib/state.rs`)

**Shared State Pattern**: Dependency injection container

```rust
pub struct AppState {
    pub database: Arc<dyn UrlDatabase>,
    pub api_key: Uuid,
    pub template_dir: String,
}
```

**Benefits**:
- **Thread Safety**: Arc allows sharing across async tasks
- **Dependency Injection**: Handlers receive dependencies through state
- **Configuration**: Runtime configuration accessible to all handlers

### 7. Error Handling (`src/lib/errors.rs`)

**Centralized Error Management**:

```rust
pub enum ApiError {
    BadRequest(String),
    NotFound(String),
    Unauthorized(String),
    // ... other variants
}
```

**Features**:
- **HTTP Status Mapping**: Errors automatically convert to appropriate HTTP codes
- **Error Chain Preservation**: Full error context maintained for debugging
- **Consistent API Responses**: All errors return JSON envelope format

### 8. Response Format (`src/lib/response.rs`)

**JSON Envelope Pattern**:

```json
{
  "success": true|false,
  "message": "descriptive message",
  "status": 200,
  "time": "2025-09-23T12:00:00Z",
  "data": { ... } | null
}
```

**Benefits**:
- **Consistent Structure**: All API responses follow same format
- **Client-Friendly**: Easy to parse and handle on frontend
- **Debugging**: Timestamps and messages aid troubleshooting

### 9. Template Engine (`src/lib/templates.rs`)

**Tera Integration**:
- **Template Compilation**: Templates compiled once at startup
- **Inheritance**: Base templates with block extensions
- **Context Injection**: Dynamic data binding
- **Static Assets**: CSS/JS served from `/static`

## 🔄 Request Flow

### URL Shortening Flow
```
1. POST /api/shorten with URL in body
2. API Key middleware validates x-api-key header
3. Handler parses and validates URL
4. Generate unique 6-character ID (nanoid)
5. Store mapping in database
6. Return shortened URL: https://host/{id}
```

### URL Redirect Flow
```
1. GET /api/redirect/{id}
2. Handler extracts ID from path
3. Database lookup for original URL
4. Return HTTP 308 Permanent Redirect
5. Browser follows redirect to original URL
```

### Admin Interface Flow
```
1. GET /admin
2. Handler loads template context
3. Tera renders HTML with base template
4. Static CSS/JS loaded from /static
5. Return complete HTML page
```

## 📊 Data Model

### Database Schema

```sql
CREATE TABLE urls (
    id TEXT PRIMARY KEY,    -- 6-character nanoid
    url TEXT NOT NULL       -- Original URL
);
```

**Design Decisions**:
- **Simple Schema**: Minimal viable product approach
- **Text IDs**: Human-readable, URL-safe identifiers
- **No Timestamps**: Keep it simple (can be added later)
- **No User Association**: Anonymous URL shortening

## 🏗️ Architectural Patterns

### 1. Layered Architecture
- **Presentation Layer**: HTTP handlers and templates
- **Business Logic Layer**: URL shortening logic
- **Data Access Layer**: Database abstraction
- **Infrastructure Layer**: Configuration, logging, startup

### 2. Dependency Injection
- **State Container**: AppState provides dependencies to handlers
- **Interface Segregation**: Small, focused traits
- **Inversion of Control**: Handlers depend on abstractions, not implementations

### 3. Repository Pattern
- **Data Access Abstraction**: UrlDatabase trait
- **Implementation Flexibility**: Easy to swap database types
- **Testing**: Mock implementations for unit tests

### 4. Command Query Separation
- **Commands**: `insert_url()` modifies state
- **Queries**: `get_url()` reads state
- **Clear Intent**: Method names indicate read vs write operations

## 🧪 Testing Architecture

### Integration Tests (`tests/api/`)

**Test Strategy**:
- **In-Memory Database**: SQLite `:memory:` for isolation
- **Full Application**: Tests complete request/response cycle
- **Helper Functions**: Shared test utilities
- **Tracing Integration**: Optional logging for debugging

**Test Structure**:
```
tests/api/
├── main.rs          # Test module declarations
├── helpers.rs       # TestApp and spawn_app()
├── health_check.rs  # Health endpoint tests
├── shorten.rs       # URL shortening tests
└── redirect.rs      # URL redirect tests
```

### Test Application Setup
```rust
pub struct TestApp {
    pub address: String,
    pub client: reqwest::Client,
    pub database: Arc<dyn UrlDatabase>,
    pub api_key: Uuid,
}
```

## 🚀 Deployment Considerations

### Configuration Management
- **Environment-Based**: Different configs for local/production
- **Environment Variables**: Override any setting via `APP_*` variables
- **Secrets Management**: API keys and sensitive data via env vars

### Database
- **File-Based**: SQLite database stored as `database.db`
- **Migrations**: Automatic migration on startup
- **Backup**: Simple file backup/restore

### Monitoring
- **Health Checks**: `/api/health_check` for load balancer probes
- **Structured Logging**: JSON logs for log aggregation
- **Request Tracing**: Correlation IDs for debugging
- **Error Handling**: Comprehensive error responses

## 🔮 Extension Points

### Database Support
The trait-based database abstraction makes it easy to add support for other databases:

```rust
pub struct PostgresUrlDatabase { /* ... */ }

impl UrlDatabase for PostgresUrlDatabase {
    // Implement trait methods for PostgreSQL
}
```

### Authentication
Current API key middleware can be extended for:
- User-based authentication
- JWT tokens  
- OAuth integration
- Rate limiting per user

### Features
The modular architecture supports adding:
- Custom short URL aliases
- URL expiration dates
- Analytics and click tracking
- User dashboards
- Bulk URL operations

### Caching
Add caching layer between handlers and database:
- Redis for distributed caching
- In-memory cache for single instance
- Cache invalidation strategies

## 📝 Development Guidelines

### Adding New Endpoints
1. Create handler in `src/lib/routes/`
2. Add route to router in `src/lib/startup.rs`
3. Add integration tests in `tests/api/`
4. Update API documentation

### Database Changes
1. Create migration files in `migrations/`
2. Update database trait if needed
3. Update SQLite implementation
4. Test migration up/down

### Configuration Changes
1. Update `Settings` structs in `configuration.rs`
2. Add to relevant YAML files
3. Update documentation
4. Add validation if needed

---

This architecture provides a solid foundation for a URL shortener service while remaining extensible and maintainable. The clear separation of concerns and trait-based abstractions make it easy for contributors to understand and extend the system.