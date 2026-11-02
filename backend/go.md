# Backend

Stack: Go 1.26 with `net/http` and `gorilla/mux`.

## Framework and Libraries

| Area        | Package                                  | Purpose                                                      |
| ----------- | ---------------------------------------- | ------------------------------------------------------------ |
| HTTP router | `github.com/gorilla/mux`                 | Routes, subrouters, path variables                           |
| CORS        | `github.com/rs/cors`                     | Allowed origins, credentials, preflight                      |
| PostgreSQL  | `github.com/jackc/pgx/v5`                | Driver behind `database/sql`; schema changes via migrations  |
| Migrations  | `github.com/pressly/goose/v3`            | SQL migrations applied at startup                            |
| Redis       | `github.com/redis/go-redis/v9`           | Cache and rate-limit counters                                |
| JWT         | `github.com/golang-jwt/jwt/v5`           | HS256 signing and verification                               |
| Validation  | `github.com/go-playground/validator/v10` | Struct tags; DTO `Validate()` for domain rules               |
| DI          | `go.uber.org/fx`                         | Composition root and lifecycle hooks                         |
| Environment | `github.com/joho/godotenv`               | `.env` loading for local development                         |
| Logging     | `log/slog`                               | Structured JSON logs                                         |
| Passwords   | `golang.org/x/crypto/bcrypt`             | Password hashing                                             |
| OAuth       | `golang.org/x/oauth2`                    | Google and Discord login                                     |
| Email       | `gopkg.in/gomail.v2`                     | SMTP delivery                                                |
| Images      | `github.com/disintegration/imaging`      | Poster and avatar resizing                                   |
| Concurrency | `golang.org/x/sync/errgroup`             | Parallel fan-out with error propagation                      |
| UUID        | `github.com/google/uuid`                 | Identifiers                                                  |
| Arrays      | `github.com/lib/pq`                      | PostgreSQL array helpers                                     |

## Dependencies

Runtime:

- `github.com/gorilla/mux`
- `github.com/rs/cors`
- `github.com/jackc/pgx/v5`
- `github.com/pressly/goose/v3`
- `github.com/redis/go-redis/v9`
- `github.com/golang-jwt/jwt/v5`
- `github.com/go-playground/validator/v10`
- `go.uber.org/fx`
- `github.com/joho/godotenv`
- `github.com/disintegration/imaging`
- `golang.org/x/sync/errgroup`
- `github.com/google/uuid`
- `github.com/lib/pq`
- `golang.org/x/crypto`
- `golang.org/x/oauth2`
- `gopkg.in/gomail.v2`

Standard library: `net/http`, `database/sql`, `context`, `encoding/json`, `log/slog`, `testing`.

Development:

- Go 1.26 toolchain — `go`, `gofmt`, `go vet`, `go test`.
- `air` — optional hot reload (`go install github.com/air-verse/air@latest`).
- `goose` CLI — optional, for creating and rolling back migrations; the server applies them through the `goose` library on startup.
- No external linter is configured. `go vet` is the lint step; if `golangci-lint` is added, run it from the root scripts.

Do not add an ORM, a second router, or another migration tool. SQL stays explicit, and schema changes stay in `migrations/`.

## Configuration

`internal/config/config.go` is the only package that reads environment variables.

- `Load()` calls `godotenv.Load()`. A missing `.env` is not an error: in production the variables come from the environment.
- Values are read with typed helpers: `getEnv`, `getEnvDuration`, `getEnvBool`. Durations use Go syntax (`15s`, `1m`); booleans accept `1/true/yes/on` and `0/false/no/off`.
- `Load()` builds `Config` and calls `Validate()`. `cmd/api/main.go` stops the process when it returns an error.
- `Validate()` rejects an empty or default `DATABASE_URL` and a `JWT_SECRET` that is empty, default, or shorter than 32 characters.
- Other packages receive `*config.Config` and never call `os.Getenv` directly.

```go
cfg, err := config.Load()
if err != nil {
    log.Fatalf("invalid configuration: %v", err)
}
```

```go
func (c *Config) Validate() error {
    var errs []error

    if c.JWTSecret == "" || c.JWTSecret == defaultJWTSecret {
        errs = append(errs, errors.New("JWT_SECRET must be set to a non-default value"))
    }
    if len(c.JWTSecret) < minJWTSecretLength {
        errs = append(errs, fmt.Errorf("JWT_SECRET must be at least %d characters long", minJWTSecretLength))
    }
    if c.DatabaseURL == "" || c.DatabaseURL == defaultDatabaseURL {
        errs = append(errs, errors.New("DATABASE_URL must be set"))
    }

    return errors.Join(errs...)
}
```

Main variables:

- Server: `SERVER_ADDRESS`, `SERVER_WRITE_TIMEOUT`, `SERVER_READ_TIMEOUT`.
- Database: `DATABASE_URL`.
- Redis: `REDIS_ADDR`, `REDIS_PASSWORD`.
- Auth: `JWT_SECRET`, `COOKIE_SECURE`.
- URLs: `FRONTEND_URL`, `BACKEND_URL`, `PARSER_URL`.
- OAuth: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REDIRECT_URL`, `SOCIAL_GOOGLE_REDIRECT_URL`, `DISCORD_CLIENT_ID`, `DISCORD_CLIENT_SECRET`, `DISCORD_CLIENT_URL`, `SOCIAL_DISCORD_CLIENT_URL`.
- Email: `KYOKUSU_EMAIL_NAME`, `KYOKUSU_EMAIL_PASSWORD`.
- Telegram: `TELEGRAM_BOT_TOKEN`, `TG_BOT_API_KEY`, `TG_CHANNEL_USERNAME`.
- Logging and proxying: `LOG_LEVEL`, `TRUST_PROXY`.

Every variable is documented in `.env.example`. Generate the JWT secret with `openssl rand -base64 48`.

## Backend Structure

```
backend/
├── cmd/
│   └── api/
│       └── main.go                 # composition root: fx.Provide, fx.Invoke, app.Run
├── internal/
│   ├── app/                        # fx wiring, mux router, CORS, HTTP server, lifecycle
│   ├── config/                     # environment loading and validation
│   ├── routes/                     # Route interface and per-feature route registration
│   ├── handlers/                   # HTTP layer (controllers)
│   ├── services/                   # business logic and orchestration
│   ├── repository/                 # SQL queries and transactions
│   ├── middleware/                 # auth, roles, rate limit, recovery, request id
│   ├── apperrors/                  # AppError and HTTP error mapping
│   ├── models/
│   │   ├── db/                     # database row structs
│   │   └── dto/                    # request/response structs and Validate()
│   ├── storage/                    # PostgreSQL and Redis clients, migrations, seeds
│   ├── lib/
│   │   └── logger/                 # slog JSON logger
│   ├── constants/                  # roles, statuses, and shared enum values
│   ├── sse/                        # Server-Sent Events hubs
│   ├── parse/                      # ranobehub parser integration
│   └── utils/                      # response, validation, cursor, files, static, conv
├── migrations/                     # goose SQL migrations
├── uploads/                        # user uploads (gitignored)
├── Dockerfile
└── go.md                           # this file
```

## Layering

The flow is `routes -> handler -> service -> repository`. In this stack the controller layer is `internal/handlers`; `internal/routes` only registers paths and middleware.

- **Routes** implement one interface and contain no logic:

  ```go
  type Route interface {
      Register(cfg *config.Config, r *mux.Router)
  }
  ```

  Each feature has `routes/<name>.go`. The constructor is registered with `app.AsRoute`, collected into `group:"routes"`, and passed to `app.NewMuxRouter`. A route entry only declares method, path, handler, and middleware.

- **Handlers** are controllers: decode the request, validate the DTO, call the service, map errors, and write the response. They contain no business logic, no SQL, and never call repositories directly. Each handler is a struct with service dependencies and `*validator.Validate`, built by `New<Name>Handler(...)`.

- **Services** hold business logic and orchestration. They depend on repository interfaces, never on concrete database implementations. Declare the interface where it is consumed (the service package) and let `*repository.<Name>Repository` satisfy it. This keeps services testable with fakes.

- **Repositories** contain SQL only. Constructors receive `*sql.DB`; methods receive `context.Context` and return database structs or DTOs. They never see `*http.Request`, `http.ResponseWriter`, or route types.

- **Transactions** belong to repositories:

  ```go
  tx, err := r.DB.BeginTx(ctx, nil)
  if err != nil {
      return err
  }
  defer tx.Rollback()

  // ... queries through tx ...

  return tx.Commit()
  ```

  When one use case must update several repositories atomically, the repository that owns the aggregate opens the transaction and passes `*sql.Tx` to the other repository methods. Handlers and services never start transactions.

- **Wiring** happens once in `cmd/api/main.go` with `fx.Provide` for config, logger, storage, repositories, services, handlers, and routes, and `fx.Invoke` for static files, background workers, cleanup hooks, and the HTTP server. `fx` is limited to `cmd/api` and `internal/app`; domain packages do not import it.

## Validation

- Every handler that accepts a body decodes it into a DTO from `internal/models/dto` and validates it before calling the service. A handler that skips validation is not done.
- Validation uses two layers:
  - `validator.Struct(dto)` for tag-based rules (`required`, `email`, `min`, `max`, and so on);
  - `dto.Validate()` for domain rules that are easier to express in Go, using helpers from `internal/utils/validation` (`EmailValidate`, `LoginValidate`, `PasswordValidate`).
- Path and query parameters are parsed and range-checked. Feed pagination uses `parseFeedPagination`: `limit` defaults to the endpoint value and must not exceed 100; `cursor` is an opaque base64 value; a non-zero `offset` is rejected in favour of the cursor.
- Malformed JSON and failed validation return `400`. The response uses the error envelope, never a bare `http.Error` string.
- Never interpolate input into SQL. All queries use placeholders (`$1`, `$2`, ...).
- Limit request bodies with `http.MaxBytesReader`; for uploads, also check the content type and size before decoding.
- API responses use DTOs. Never expose password hashes, verification tokens, reset tokens, or raw database rows.

## Error Layer

`internal/apperrors` is the single error layer.

- `AppError` carries a stable machine code, an HTTP status, a safe message, and an optional wrapped cause:

  ```go
  type AppError struct {
      Code    string
      Message string
      Status  int
      Err     error
  }

  func (e *AppError) Error() string { return e.Message }
  func (e *AppError) Unwrap() error { return e.Err }
  ```

- Define sentinels for expected cases (`ErrNotFound`, `ErrForbidden`, `ErrConflict`, `ErrValidation`, `ErrUnauthorized`) and wrap them with context. Use `errors.Is`/`errors.As` instead of string comparison.
- `apperrors.Write(w, r, err)` maps an error to the response:
  - `*AppError` — its status and code;
  - `sql.ErrNoRows` — `404 NOT_FOUND`;
  - PostgreSQL unique violation (`23505`) — `409 CONFLICT`;
  - context cancellation — return without writing if the client is already gone;
  - anything else — `500 INTERNAL_ERROR`, logged with the request id.
- Envelope, identical to the TypeScript stack:

  ```json
  {
    "code": "VALIDATION_ERROR",
    "message": "invalid request data",
    "requestId": "6f1c...",
    "timestamp": "2026-11-02T12:00:00Z"
  }
  ```

- Status mapping:

  | Case                         | Status | Code               |
  | ---------------------------- | ------ | ------------------ |
  | Malformed JSON or validation | 400    | `VALIDATION_ERROR` |
  | Missing or invalid token     | 401    | `UNAUTHORIZED`     |
  | Insufficient role            | 403    | `FORBIDDEN`        |
  | Missing row                  | 404    | `NOT_FOUND`        |
  | Unique or state conflict     | 409    | `CONFLICT`         |
  | Rate limit                   | 429    | `RATE_LIMITED`     |
  | Unexpected failure           | 500    | `INTERNAL_ERROR`   |

- Log `4xx` at `warn` and `5xx` at `error` with the request id. Never return `err.Error()` for a `500`: it can contain SQL, file paths, or secrets.
- The panic-recovery middleware logs the stack and writes the same `500` envelope.
- Handlers use the shared `response` helpers for success payloads and `apperrors.Write` for failures. Do not mix plain-text errors and JSON in the same API.

## Authentication

- JWT with `github.com/golang-jwt/jwt/v5`, algorithm `HS256` only. The secret comes from `JWT_SECRET` and is validated at startup (at least 32 characters, not the default).
- Claims embed `jwt.RegisteredClaims` and add the application fields:

  ```go
  type Claims struct {
      UserID int    `json:"user_id"`
      Email  string `json:"email"`
      Role   string `json:"role"`
      jwt.RegisteredClaims
  }
  ```

- `GenerateJWT` sets `iat` and `exp`. The parser must reject every algorithm except `HS256` in the key function; never trust `token.Header["alg"]` without checking it.
- The token lives in an httpOnly cookie. Keep the name in one constant; KyokusuLib uses `KYOKUSU_API_TOKEN`.
  - `HttpOnly: true`;
  - `Secure: cfg.CookieSecure` (`true` in production, behind HTTPS);
  - `SameSite: http.SameSiteLaxMode`;
  - `Path: "/"`;
  - `Expires: time.Now().AddDate(0, 1, 0)`.
- Logout sets `MaxAge: -1` and `Expires: time.Unix(0, 0)` with the same flags. Never return the token in a JSON body or store it in `localStorage`.
- Middleware:
  - `AuthMiddleware(next, secret)` requires a valid cookie and puts `userID`, `email`, and `role` into the request context with typed keys;
  - `DefaultMiddleware(next, secret)` allows anonymous requests but populates the context when a valid cookie is present;
  - `RoleGuard(next, required)` enforces the hierarchy `user < publisher < moderator < admin`.
- The reference implementation issues stateless JWTs. Logout clears the cookie and marks the user offline, but an already-issued token remains valid until it expires. If revocation is required, add `jti` to the claims and a session or denylist table checked by the middleware.
- Cookie-authenticated mutations must pass an `Origin` check against `FRONTEND_URL` in addition to `SameSite=Lax`. CORS is not a CSRF defence on its own.
- Set `COOKIE_SECURE=true` whenever the API is served over HTTPS.

## Migrations

Migrations use `pressly/goose/v3` with plain SQL files.

- Directory: `backend/migrations/`.
- File name: `<yyyymmddHHMMSS>_<snake_case_name>.sql`.
- Template:

  ```sql
  -- +goose Up
  -- +goose StatementBegin
  CREATE TABLE example (
      id BIGSERIAL PRIMARY KEY
  );
  -- +goose StatementEnd

  -- +goose Down
  -- +goose StatementBegin
  DROP TABLE example;
  -- +goose StatementEnd
  ```

- `storage.NewPostgresDB` sets the goose dialect to `postgres` and runs `goose.Up(db, "migrations")` before the HTTP server starts. Applied versions are tracked in `goose_db_version`.
- `Down` is mandatory and must restore the previous schema.
- Never edit an already applied migration — create a new one. Never delete or rewrite user data in a migration without an explicit request.
- Create, inspect, and roll back with the goose CLI:

  ```bash
  goose -dir migrations create <snake_case_name> sql
  goose -dir migrations postgres "$DATABASE_URL" status
  goose -dir migrations postgres "$DATABASE_URL" up
  goose -dir migrations postgres "$DATABASE_URL" down
  ```

- Reference data is seeded after migrations by `storage.runSeeds`, using `INSERT ... ON CONFLICT DO NOTHING` so the operation is idempotent. Seeds cover fixed dictionaries such as tags, levels, genres, and categories. Everything else belongs in a migration.
- Extensions are created with `CREATE EXTENSION IF NOT EXISTS`.
- The Docker image copies `migrations/`, so the same startup path works in production.

## Tests

- Tests use the standard `testing` package and live next to the code as `*_test.go`.
- Prefer table-driven tests with `t.Run`.
- Handler tests use `httptest.NewRequest` and `httptest.NewRecorder`; construct the handler with a real service or a fake. Cookie tests assert `HttpOnly`, `Secure`, `SameSite`, `Path`, `Expires`, and the logout flags.
- Middleware tests generate a real token with `service.GenerateJWT` and attach it as a cookie. Benchmarks use `b.ReportAllocs()` and `b.ResetTimer()`.
- Config tests use `t.Setenv` and `t.Chdir(t.TempDir())` so a developer's `.env` cannot affect the result.
- Integration tests are opt-in:
  - `TEST_DATABASE_URL` — service and repository tests; the test skips when it is unset;
  - `BENCH_DATABASE_URL` — benchmarks.
  Run them against a disposable PostgreSQL database, never against production.
- Repository tests run against a real PostgreSQL with migrations applied. Keep each test isolated with a transaction rollback or a dedicated test database.
- CORS tests build the handler from `corsOptions` and check allowed origin, foreign origin, credentials, and preflight.
- Run `go test ./...`. Report coverage from `go test -coverprofile=coverage.out ./...` and `go tool cover -func=coverage.out`.

## Logging and Request Context

- `internal/lib/logger` builds `slog.New(slog.NewJSONHandler(os.Stdout, ...))`; the level comes from `LOG_LEVEL` and defaults to `info`.
- A request-id middleware accepts `X-Request-Id` or generates a UUID, stores it in the context, returns it in the response header, and includes it in every log line and error envelope.
- The request logging middleware records method, path, status, duration, remote address, and request id. Wrap `http.ResponseWriter` to capture the status code.
- The panic-recovery middleware logs the panic and stack at `error` level and returns the standard `500` envelope.
- Never log passwords, tokens, cookies, authorization headers, or full request bodies.

## Rate Limiting and CORS

- The limiter is Redis-backed and fixed-window: a Lua script runs `INCR` and sets `PEXPIRE` on the first hit. The key is `rate_limit:<path>:<clientIP>`.
- The default is 60 requests per minute, applied to sensitive POST routes such as login, registration, password reset, and comment creation.
- A rejected request returns `429` with `{ "code": "RATE_LIMITED", "message": "..." }`.
- Redis is optional for caching: `storage.NewRedisClient` returns `nil` when the server is unreachable. The limiter and cache must nil-check the client and fail open with a warning. Do not treat rate limiting as a security boundary when Redis is down.
- Resolve the client IP from `r.RemoteAddr`. Behind a trusted reverse proxy, read the first `X-Forwarded-For` address only when `TRUST_PROXY=true`; never trust that header from an untrusted client.
- CORS uses `rs/cors`:
  - allowed origin: `FRONTEND_URL`, with the trailing slash removed;
  - methods: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`;
  - headers: `Content-Type`, `Authorization`, `X-Requested-With`;
  - exposed headers: `X-Next-Cursor`, `X-Has-More`;
  - `AllowCredentials: true`, `MaxAge: 600`.
- Cookies require an exact origin and credentials; wildcard origins are forbidden.

## Graceful Shutdown and Health

- `cmd/api/main.go` creates the fx application, checks `app.Err()`, then calls `app.Run()`. fx listens for `SIGINT`/`SIGTERM` and runs the `OnStop` hooks.
- `app.StartHTTPServer` starts `http.Server` in a goroutine and calls `srv.Shutdown(ctx)` on stop. Server timeouts come from `SERVER_WRITE_TIMEOUT` and `SERVER_READ_TIMEOUT`; set `ReadHeaderTimeout` and `IdleTimeout` explicitly.
- `storage.NewPostgresDB` closes `*sql.DB` on stop; `app.RegisterRedisCleanup` closes the Redis client; background workers and SSE hubs are cancelled through their context and closed in `OnStop` hooks.
- `GET /api/health` is the liveness probe and returns status, timestamp, and uptime. Add `GET /api/ready` that pings PostgreSQL (and Redis when it is required) and returns `503` when a dependency is unavailable.
- The `scratch` runtime image has no shell or `curl`. If compose needs a backend healthcheck, use an external probe or switch the runtime stage to an image that contains a health binary.

## Go and Tooling

- Go 1.26; the module path is declared in `backend/go.mod`. Commit `go.mod` and `go.sum`; do not vendor dependencies.
- Format: `gofmt -w .`; check with `gofmt -l .` (the command must print nothing).
- Vet: `go vet ./...`.
- Build: `go build ./...`; produce a binary with `go build -o bin/api ./cmd/api`.
- Test: `go test ./...`; race detector with `go test -race ./...`; coverage with `go test -coverprofile=coverage.out ./...`.
- Hot reload in development: `air` from `backend/`.
- The root scripts from `../AGENTS.md` should wrap these commands so the whole workspace is checked with one command.

## Docker

- Multi-stage build:
  - `golang:1.26-alpine AS deps` — copy `go.mod` and `go.sum`, run `go mod download`;
  - `builder` — copy the module cache and source, build with `CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o main ./cmd/api`;
  - `scratch` — copy the binary, `migrations/`, and CA certificates.
- `docker-compose.yml` builds the service with `context: ./backend` and `dockerfile: Dockerfile`, and waits for PostgreSQL and Redis healthchecks.
- `EXPOSE 8080`; the listen address comes from `SERVER_ADDRESS`.
- Mount uploads as a volume (`./backend/uploads:/app/uploads`) so files survive container restarts.
- The image must contain `migrations/`; migrations run automatically on startup.
- The `scratch` stage runs as root by default. For production, set a non-root `USER` and make the uploads volume writable by that UID.

## Commands

Run from `backend/`:

```bash
go run ./cmd/api
go build -o bin/api ./cmd/api
go test ./...
go test -race ./...
go test -coverprofile=coverage.out ./...
go tool cover -func=coverage.out
gofmt -l .
go vet ./...
```

Migrations:

```bash
goose -dir migrations create <snake_case_name> sql
goose -dir migrations postgres "$DATABASE_URL" status
goose -dir migrations postgres "$DATABASE_URL" up
goose -dir migrations postgres "$DATABASE_URL" down
```

See the Toolchain section of `../AGENTS.md` for the root scripts that wrap these commands.
