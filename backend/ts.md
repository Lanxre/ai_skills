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
