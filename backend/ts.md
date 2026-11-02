# Backend

Stack: TypeScript on Bun.

## Framework and Libraries

| Area          | Package             | Purpose                                            |
| ------------- | ------------------- | -------------------------------------------------- |
| HTTP server   | `elysia`            | Backend server and routing                         |
| CORS          | `@elysia/cors`      | Elysia plugin for CORS headers                     |
| Rate limiting | `elysia-rate-limit` | Elysia plugin for request rate limiting            |
| PostgreSQL    | `postgres`          | SQL client; schema changes only through migrations |
| JWT           | `jose`              | Signing and verifying JSON Web Tokens              |
| Validation    | `valibot`           | Environment configuration validation               |
| Redis         | Bun built-in `Bun.redis` | Redis client built into Bun, no external dependency |

## Dependencies

Runtime:

- `elysia`
- `@elysia/cors`
- `elysia-rate-limit`
- `postgres`
- `jose`
- `valibot`

Development:

- `bun-types` — Bun typings for this layer
- `oxfmt` — formatter, managed at the workspace root
- `oxlint` — linter, managed at the workspace root

Do not add a separate Redis client (`redis`, `ioredis`, and so on): use the client built into Bun.

## Configuration

All environment variables are parsed once in `src/config` with valibot and validated at startup: a missing or invalid variable must fail the process immediately.

```ts
import * as v from "valibot";

const EnvSchema = v.object({
  NODE_ENV: v.optional(v.picklist(["development", "test", "production"]), "development"),
  PORT: v.optional(v.pipe(v.string(), v.transform(Number), v.number(), v.integer()), 3000),
  DATABASE_URL: v.pipe(v.string(), v.minLength(1)),
  REDIS_URL: v.pipe(v.string(), v.minLength(1)),
  JWT_SECRET: v.pipe(v.string(), v.minLength(32)),
  CORS_ORIGINS: v.optional(v.string(), ""),
  LOG_LEVEL: v.optional(v.picklist(["debug", "info", "warn", "error"]), "info"),
});

export const config = v.parse(EnvSchema, Bun.env);
```

- Read `Bun.env` only inside `src/config`; other modules import the typed `config`.
- Every required variable is documented in `.env.example`.
- Main variables: `NODE_ENV`, `PORT`, `HOST`, `DATABASE_URL`, `DATABASE_POOL_MAX`, `REDIS_URL`, `JWT_SECRET`, `JWT_TTL_SECONDS`, `CORS_ORIGINS`, `RATE_LIMIT_WINDOW_MS`, `RATE_LIMIT_MAX`, `RATE_LIMIT_AUTH_MAX`, `TRUST_PROXY`, `LOG_LEVEL`.

## Backend Structure

```
backend/
├── src/
│   ├── modules/
│   │   └── <name>/
│   │       ├── <name>.controller.ts   # HTTP only: routes, validation, status codes
│   │       ├── <name>.service.ts      # business logic, depends on the repository interface
│   │       ├── <name>.repository.ts   # SQL only: <Name>Repository + Pg<Name>Repository
│   │       └── <name>.model.ts        # Elysia schemas and derived TypeScript types
│   ├── shared/
│   │   ├── database/                  # PostgreSQL client and migration runner
│   │   ├── errors/                    # AppError and HTTP status mapping
│   │   ├── http/                      # cookies, request id, response envelope
│   │   ├── logger/                    # JSON logger
│   │   └── redis/                     # Bun.redis client
│   ├── config/                        # valibot env schema
│   ├── scripts/                       # migration CLI (create/up/down/remove)
│   ├── workers/                       # background jobs
│   ├── app.ts                         # Elysia app: plugins, middleware, onError
│   └── server.ts                      # composition root: wiring, listen, shutdown
├── migrations/                        # <uuid>_<name>_<yyyymmddHHMMSS>.sql
├── test/                              # *.test.ts, setup.ts, db.ts
├── bunfig.toml                        # test preload
├── Dockerfile
└── ts.md                              # this file
```

## Layering

The flow is `controller -> service -> repository`.

- Controllers handle HTTP only: routing, Elysia schemas, status codes, and error mapping. No business logic, no SQL, no repository calls.
- Services hold business logic and orchestration. They depend on repository interfaces, never on concrete implementations.
- Repositories own all data access, including transactions: a repository method opens `sql.begin(...)` and never sees `Request`, `Response`, or Elysia types. When a use case must update several repositories atomically, the repository that owns the aggregate opens the transaction and passes the transaction handle to the other repositories.
- Controllers are factories: `create<Name>Controller(service)` returns an Elysia instance.
- Services and repositories receive dependencies through constructors: `new <Name>Service(repository, ...)`, `new Pg<Name>Repository(sql)`.
- Wire everything once in the composition root (`src/server.ts`). Do not create repositories or database clients inside services or controllers.

## Validation

- Every route declares schemas for the input it accepts (`params`, `query`, and/or `body`) and a `response` schema. A route without a `response` schema is not done.
- HTTP schemas use Elysia's built-in `t` (TypeBox). Valibot is reserved for environment configuration.
- Elysia `VALIDATION` errors map to `422`, malformed payloads (`PARSE`) to `400`.

## Error Layer

- `src/shared/errors` defines the base `AppError` with `code`, `status`, and `message`. Module-specific errors extend it (for example `AuthError` with an auth code).
- One `onError` handler in `src/app.ts` maps errors to HTTP responses:
  - `AppError` — the status declared by the error;
  - Elysia `VALIDATION` — `422`;
  - Elysia `PARSE` — `400`;
  - Elysia `NOT_FOUND` — `404`;
  - rate limit — `429`;
  - anything else — `500`, without leaking stack traces or SQL.
- Error envelope: `{ code, message, requestId, timestamp }`.
- Log `4xx` at `warn` and `5xx` at `error` with the request id.

## Authentication

- JWT with `jose`, algorithm `HS256`, secret from `JWT_SECRET` (at least 32 characters).
- Claims: `sub` (user id), `jti` (session id), `iat`, `exp`. Verify with `jwtVerify(token, key, { algorithms: ["HS256"] })`.
- The token is stored in an httpOnly cookie: `httpOnly: true`, `secure` in production, `sameSite: "lax"`, `path: "/"`. Never put it in `localStorage` or a JSON response body.
- Session TTL is defined by `JWT_TTL_SECONDS`; remember-me sets `maxAge`, otherwise the cookie is a session cookie.
- Store `sha256(token)` in the `sessions` table and resolve the user by `jti`; revocation deletes the row. Purge expired sessions periodically.
- Mutations that carry auth cookies require a trusted `Origin` (CSRF protection); the allowed origins come from `CORS_ORIGINS`.

## Migrations

Custom runner, no external migration dependency.

- Core: `src/shared/database/migrations.ts` — `createMigration`, `acceptMigration`, `denyMigration`, `removeMigration`, `applyLocalMigration`.
- CLI: `src/scripts/create-migration.ts`, `accept-migration.ts`, `deny-migration.ts`, `remove-migration.ts`.
- Scripts in `backend/package.json`: `migrate:create <name>`, `migrate:up`, `migrate:down`, `migrate:remove`.
- File name: `migrations/<uuid>_<name>_<yyyymmddHHMMSS>.sql`.
- File template:

  ```
  -- Migration: <name> Up
  -- Migration StateBegin

  -- Migration StatementEnd

  -- Migration: <name> Down
  -- Migration StateBegin

  -- Migration StatementEnd
  ```

- `_migrations` table: `id uuid primary key`, `name varchar unique`, `created_at timestamptz`.
- `migrate:create` inserts a `_migrations` row and creates the file. Fill in both `Up` and `Down`, then apply with `migrate:up`, which executes the `Up` block of the latest `_migrations` row. Run it once per created migration.
- `migrate:down` executes `Down` and deletes the `_migrations` row in one transaction.
- `migrate:remove` deletes the latest row and its file.
- On startup, `applyLocalMigration` applies files whose names are missing from `_migrations` (for example, migrations merged from another branch), each in a transaction.
- Never edit an already applied migration — create a new one. `Down` is mandatory and must restore the previous schema.
- Extensions are created with `CREATE EXTENSION IF NOT EXISTS`.

## Tests

- Tests live in `backend/test/*.test.ts`; `bunfig.toml` preloads `test/setup.ts`.
- `test/setup.ts` sets `NODE_ENV=test`, silences logs, and defines `TEST_DATABASE_URL` (or derives `<DATABASE_URL>_test`), plus test-only secrets and generous rate limits.
- `test/db.ts` creates the test database if needed, runs `applyLocalMigration`, returns a singleton client with `max: 1`, and exposes `resetData()` (`TRUNCATE ... CASCADE`) and `closeTestDb()`.
- Repositories are tested against a real PostgreSQL; services are tested with fakes.
- Contract tests call the Elysia app directly (`app.handle(new Request(...))`).
- Run `bun run test` and report coverage from `bun run test:coverage`.

## Logging and Request Context

- JSON logger in `src/shared/logger` with levels from `LOG_LEVEL`.
- `src/shared/http/request-context.ts` assigns a request id from `x-request-id` or a UUID and returns it in the response header.
- `onAfterResponse` logs method, path, status, duration, and request id; `onError` logs failures with the same id.

## Rate Limiting and CORS

- `elysia-rate-limit` with the window and limits from environment variables; the key is the client IP.
- `TRUST_PROXY=true` only behind a trusted reverse proxy that sets `X-Forwarded-For`.
- The rate-limit response is `429` with `{ code: "RATE_LIMITED", message }`.
- Auth endpoints use a stricter limiter (`RATE_LIMIT_AUTH_MAX`) than public routes.
- CORS origins come from `CORS_ORIGINS`; enable `credentials` because authentication uses cookies.

## Graceful Shutdown and Health

- `src/server.ts` builds the app, starts the server and workers, and on `SIGINT`/`SIGTERM` closes PostgreSQL and Redis before exiting with code `0`.
- `GET /api/health` is a liveness probe; `GET /api/ready` pings PostgreSQL and returns `503` when the database is unavailable.

## TypeScript and Tooling

- `tsconfig.json`: `strict`, `noUncheckedIndexedAccess`, `moduleResolution: "bundler"`, `types: ["bun"]`, import alias `@/*` → `./src/*`.
- Formatter: `oxfmt` via `bun run format`, check with `bun run format:check`.
- Linter: `oxlint` via `bun run lint`.
- Typecheck: `tsc --noEmit` via `bun run typecheck`.
- Tests: `bun run test`, coverage via `bun run test:coverage`.

## Docker

- Base image `oven/bun:1-alpine`, multi-stage build: install dependencies with `bun install --frozen-lockfile --production`, then copy `backend/`.
- Build context is the repository root; `docker-compose.yml` builds the backend service from `backend/Dockerfile`.
- Run as `bun run src/server.ts`, expose the configured `PORT`.

## Commands

Run from the repository root. See the Toolchain section of `../AGENTS.md` for the full list of root scripts. Migration commands run from `backend/` with the environment file:

```bash
bun run migrate:create <snake_case_name>
bun run migrate:up
bun run migrate:down
bun run migrate:remove
```
