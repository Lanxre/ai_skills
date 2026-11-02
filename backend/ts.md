# Backend

Stack: TypeScript on Bun.

## Framework and Libraries

| Area          | Package                | Purpose                                              |
| ------------- | ---------------------- | ---------------------------------------------------- |
| HTTP server   | `elysia`               | Backend server and routing                           |
| CORS          | `@elysiajs/cors`       | Elysia plugin for CORS headers                       |
| Rate limiting | `elysia-rate-limit`    | Elysia plugin for request rate limiting              |
| PostgreSQL    | `postgres`             | SQL client; schema changes only through migrations   |
| JWT           | `jose`                 | Signing and verifying JSON Web Tokens                |
| Redis         | Bun built-in `Bun.redis` | Redis client built into Bun, no external dependency |

## Dependencies

Runtime:

- `elysia`
- `@elysiajs/cors`
- `elysia-rate-limit`
- `postgres`
- `jose`

Development:

- `bun-types` — Bun typings for this layer
- `oxfmt` — formatter, managed at the workspace root
- `oxlint` — linter, managed at the workspace root

Do not add a separate Redis client (`redis`, `ioredis`, and so on): use the client built into Bun.

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
│   │   ├── database/                  # PostgreSQL client
│   │   ├── http/                      # response helpers, error mapping
│   │   └── redis/                     # Bun.redis client
│   ├── config/                        # environment parsing and validation
│   ├── workers/                       # background jobs
│   └── server.ts                      # composition root: repositories -> services -> controllers
├── migrations/                        # SQL migrations
└── ts.md                              # this file
```

- Controllers are factories: `create<Name>Controller(service)` returns an Elysia instance.
- Services receive their dependencies through the constructor: `new <Name>Service(repository, ...)`.
- Repositories receive the PostgreSQL client through the constructor: `new Pg<Name>Repository(sql)`.
- Wire repositories, services, and controllers once in the composition root (`src/server.ts`). Do not create repositories inside services or controllers.

## Tooling

- Runtime and package manager: Bun (workspace root: `../package.json`)
- Formatter: `oxfmt` via `bun run format`, check with `bun run format:check`
- Linter: `oxlint` via `bun run lint`
- Typecheck: `tsc --noEmit` via `bun run typecheck`
- Tests: `bun run test`, coverage via `bun run test:coverage`

## Configuration

- PostgreSQL connection and Redis connection are configured through environment variables and documented in `.env.example`.
- Never hardcode credentials; see the Security section of `../AGENTS.md`.

## Commands

Run from the repository root. See the Toolchain section of `../AGENTS.md` for the full list of scripts.
