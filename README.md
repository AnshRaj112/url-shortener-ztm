# URL Shortener

A high-performance URL shortener service built with modern Rust technologies. This service provides a simple API for creating shortened URLs and redirecting users to their original destinations.

## 🚀 Features

- **Fast URL shortening**: Generate short, unique identifiers for long URLs using nanoid
- **Reliable redirects**: Permanent redirects to original URLs with proper HTTP status codes
- **Rate limiting**: Built-in rate limiting to prevent abuse using tower-governor
- **SQLite storage**: Lightweight, file-based database with automatic migrations
- **Database abstraction**: Trait-based database layer for easy database switching
- **Comprehensive logging**: Structured logging with tracing and request IDs
- **Health monitoring**: Built-in health check endpoint
- **Web interface**: Admin panel with Tera templates
- **API key protection**: Secure API endpoints with UUID-based authentication
- **Production ready**: Built for deployment with graceful shutdown handling

## 🛠 Technology Stack

- **Framework**: [Axum](https://github.com/tokio-rs/axum) - Modern async web framework
- **Database**: SQLite with [SQLx](https://github.com/launchbadge/sqlx) for type-safe queries
- **Rate Limiting**: [tower-governor](https://crates.io/crates/tower_governor) - Per-IP rate limiting with GCRA algorithm
- **Templates**: [Tera](https://keats.github.io/tera/) - Template engine for web interface
- **Configuration**: [Figment](https://github.com/SergioBenitez/figment) - Layered configuration
- **Logging**: Structured logging with `tracing` and Bunyan formatting
- **Testing**: Comprehensive integration tests with in-memory SQLite databases

## 📡 API Endpoints

### Shorten a URL
```bash
POST /api/shorten
Content-Type: text/plain
x-api-key: YOUR_API_KEY

# Example
curl -d 'https://www.google.com/' \
  -H "x-api-key: e4125dd1-3d3e-43a1-bc9c-dc0ba12ad4b5" \
  http://localhost:8000/api/shorten
```

**Response**: Returns the shortened URL
```
https://localhost:8000/AbC123
```

### Redirect to Original URL
```bash
GET /api/redirect/{id}

# Example
curl -L http://localhost:8000/api/redirect/AbC123
```

**Response**: HTTP 308 Permanent Redirect to the original URL

### Health Check
```bash
GET /api/health_check

# Example
curl http://localhost:8000/api/health_check
```

**Response**: HTTP 200 OK with JSON envelope
```json
{
  "success": true,
  "message": "ok",
  "status": 200,
  "time": "2025-09-18T12:00:00Z",
  "data": null
}
```

### Admin Interface
```bash
GET /admin

# Example - View the web interface
curl http://localhost:8000/admin
```

**Response**: HTML page with admin interface

## 🚀 Quick Start

### Prerequisites
- [Rust](https://rustup.rs/) (latest stable)
- [SQLx CLI](https://crates.io/crates/sqlx-cli)
- No external database required (uses SQLite)

### Local Development

1. **Clone the repository**
   ```bash
   git clone https://github.com/yourusername/url-shortener-ztm.git
   cd url-shortener-ztm
   ```

2. **Install dependencies**
   ```bash
   cargo build
   ```

3. ** Create the Database**
  ```bash
  sqlx database create
  ```

4. **Run the application**
   ```bash
   cargo run
   ```
   
   The database and migrations will be set up automatically on first run.

5. **Test the service**
   ```bash
   # Get your API key from configuration/base.yaml (or set via environment)
   API_KEY="e4125dd1-3d3e-43a1-bc9c-dc0ba12ad4b5"
   
   # Shorten a URL
   curl -d 'https://example.com' \
     -H "x-api-key: $API_KEY" \
     http://localhost:8000/api/shorten
   
   # Visit the shortened URL
   curl -L http://localhost:8000/api/redirect/AbC123
   
   # Check health
   curl http://localhost:8000/api/health_check
   
   # Visit admin interface
   open http://localhost:8000/admin
   ```

### Configuration

The application supports environment-based configuration with YAML files:

#### Configuration Files
- `configuration/base.yaml` - Base configuration
- `configuration/local.yaml` - Local development overrides
- `configuration/production.yaml` - Production settings

#### Environment Variables
Set `APP_ENVIRONMENT` to `local` or `production` to load the appropriate config.

Override any setting using environment variables with `APP_` prefix:
```bash
APP_APPLICATION__PORT=3000
APP_APPLICATION__API_KEY=your-new-api-key
APP_DATABASE__DATABASE_PATH=./my-database.db
```

#### Database Configuration
```yaml
database:
  database_path: "sqlite:database.db"  # Path to SQLite database file
  create_if_missing: true              # Create database if it doesn't exist
```

For in-memory database (testing):
```yaml
database:
  database_path: ":memory:"
  create_if_missing: true
```

#### Rate Limiting Configuration
The service includes built-in rate limiting to prevent abuse using the [tower-governor](https://crates.io/crates/tower_governor) crate:

```yaml
rate_limiting:
  enabled: true              # Enable/disable rate limiting
  requests_per_second: 10    # Maximum sustained request rate per IP
  burst_size: 5              # Additional burst capacity per IP
```

**Environment-specific examples:**

**Development** (`configuration/local.yaml`):
```yaml
rate_limiting:
  enabled: true
  requests_per_second: 20    # More lenient for development
  burst_size: 10
```

**Production** (`configuration/production.yaml`):
```yaml
rate_limiting:
  enabled: true
  requests_per_second: 5     # Strict rate limiting for production
  burst_size: 3
```

**Rate Limiting Behavior:**
- Limits are applied **per IP address** using the GCRA (Generic Cell Rate Algorithm)
- Only **URL shortening endpoints** are rate limited (`/api/shorten`, `/api/public/shorten`)
- Health checks and redirects are **not rate limited**
- Standard HTTP headers are included in rate limit responses:
  - `retry-after`: Seconds to wait before retrying
  - `x-ratelimit-after`: Additional rate limiting information
- Returns **HTTP 429 Too Many Requests** when limits are exceeded

**Environment Variable Override:**
```bash
APP_RATE_LIMITING__ENABLED=false                 # Disable rate limiting
APP_RATE_LIMITING__REQUESTS_PER_SECOND=100       # 100 requests per second
APP_RATE_LIMITING__BURST_SIZE=20                 # Allow bursts of 20 requests
```

## 🧪 Testing

The project includes comprehensive integration tests using in-memory SQLite databases.

```bash
# Run all tests
cargo test

# Run tests with logging output
TEST_LOG=1 cargo test

# Run specific test module
cargo test health_check
cargo test redirect
cargo test shorten
```

### Test Coverage
- ✅ Health check endpoint with JSON envelope validation
- ✅ URL shortening functionality with API key authentication
- ✅ URL redirection with proper HTTP status codes
- ✅ Rate limiting with per-IP enforcement and proper HTTP headers
- ✅ Database integration with trait abstraction
- ✅ Error handling and edge cases

## 🏗 Project Structure

```
src/
├── bin/
│   └── main.rs                    # Application entry point
└── lib/
    ├── lib.rs                     # Library crate root
    ├── configuration.rs           # Configuration management
    ├── errors.rs                  # Error types and handling
    ├── middleware.rs              # API key authentication
    ├── response.rs                # JSON response envelope
    ├── startup.rs                 # Application startup and router
    ├── state.rs                   # Application state management
    ├── telemetry.rs               # Logging and tracing setup
    ├── templates.rs               # Template rendering
    ├── database/
    │   ├── mod.rs                 # Database trait definitions
    │   └── sqlite.rs              # SQLite implementation
    └── routes/
        ├── mod.rs                 # Route module exports
        ├── health_check.rs        # Health check handler
        ├── index.rs               # Admin interface handler
        ├── shorten.rs             # URL shortening handler
        └── redirect.rs            # URL redirect handler

tests/
└── api/
    ├── main.rs                    # Integration test entry
    ├── helpers.rs                 # Test utilities and setup
    ├── health_check.rs            # Health check tests
    ├── shorten.rs                 # URL shortening tests
    └── redirect.rs                # URL redirect tests

configuration/
├── base.yaml                      # Base configuration
├── local.yaml                     # Local development config
└── production.yaml                # Production config

migrations/
├── 20250917043645_url_shortener_ztm.up.sql    # Database schema
└── 20250917043645_url_shortener_ztm.down.sql  # Rollback migration

static/                            # Static web assets
├── screen.css                     # CSS styles
└── scripts.js                     # JavaScript

templates/                         # Tera templates
├── base.html                      # Base template
└── index.html                     # Admin interface
```

## 🔧 Architecture

### Database Layer
The application uses a trait-based database abstraction (`UrlDatabase`) that currently supports SQLite but can be easily extended to other databases:

```rust
#[async_trait]
pub trait UrlDatabase: Send + Sync {
    async fn insert_url(&self, id: &str, url: &str) -> Result<(), DatabaseError>;
    async fn get_url(&self, id: &str) -> Result<String, DatabaseError>;
}
```

### Error Handling
Comprehensive error handling with custom `ApiError` types and structured JSON responses:

```rust
pub enum ApiError {
    BadRequest(String),
    NotFound(String),
    Unauthorized(String),
    Internal(String),
    // ...
}
```

### Configuration Management
Layered configuration system supporting YAML files and environment variables with automatic environment detection.

## 📊 Database Schema

```sql
CREATE TABLE urls (
  id TEXT PRIMARY KEY,              -- Short identifier (nanoid, 6 characters)
  url TEXT NOT NULL                 -- Original URL
);
```

## 🔍 Monitoring & Observability

- **Structured Logging**: JSON-formatted logs with request correlation IDs
- **Request Tracing**: Full request lifecycle tracing with `tracing` crate
- **Health Checks**: `/api/health_check` endpoint with JSON envelope response
- **Error Handling**: Comprehensive error responses with appropriate HTTP status codes
- **Request IDs**: Automatic request ID generation and propagation

## 🔒 Security

- **API Key Authentication**: Protected endpoints require valid UUID-based API keys
- **Input Validation**: URL parsing and validation before storage
- **SQL Injection Protection**: Type-safe queries with SQLx
- **Error Information Disclosure**: Sanitized error responses

## 🚧 Roadmap

- [x] SQLite database support with migrations
- [x] Database abstraction layer
- [x] Web UI with Tera templates
- [x] API key authentication
- [x] Comprehensive error handling
- [x] Integration tests
- [x] PostgreSQL database support
- [ ] User authentication and URL management
- [ ] Analytics and usage statistics
- [ ] Custom short URL aliases
- [ ] URL expiration and cleanup
- [ ] Rate limiting
- [ ] Docker containerization

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

### Development Guidelines
- Ensure all tests pass (`cargo test`)
- Follow Rust naming conventions
- Add tests for new functionality
- Update documentation as needed

## 📄 License

This project is licensed under the MIT License - see the [License.txt](License.txt) file for details.

## 👤 Author

**Jeffery D. Mitchell**
- Email: crusty.rustacean@gmail.com
- GitHub: [@crustyrustacean](https://github.com/crustyrustacean)

## 🙏 Acknowledgments

- Built with the excellent Rust web ecosystem
- Inspired by modern web service architecture patterns
- Thanks to the Rust community for amazing tools and libraries