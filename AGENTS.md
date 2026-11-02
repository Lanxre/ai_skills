# App Development Guidelines

1. Follow the SOLID principles. Concretely: keep domain logic out of controllers and UI components, depend on interfaces rather than implementations, and inject dependencies instead of constructing them inline.
2. Prefer composition (aggregation) and delegation over inheritance. Do not build deep inheritance chains.
3. After every change or addition, analyze the project as described in [Post-Change Analysis](#post-change-analysis) before finishing or committing.

## Architecture

The application is split into two main layers, `backend` and `frontend`, plus standalone services.

- Before writing code, read the stack description for the active stack. Keep one file per stack, named after the stack: `backend/<stack>.md` and `frontend/<stack>.md` (for example `backend/ts.md`, `frontend/vue.md`). If no file describes the stack you need, stop and ask the user.
- Standalone runnable services (for example, a Telegram bot) go into `apps/`. Only self-contained applications belong there, and each has its own `README.md`.
- Project documentation and specifications go into `docs/`. The only exceptions are `README.md` and `AGENTS.md` in the repository root, `backend/*.md` / `frontend/*.md` for stack descriptions, and `apps/*/README.md` for standalone apps.
- Do not add new top-level directories without asking the user.
- All packages live in a single Bun workspace rooted at the repository root; all commands are run from there (see [Toolchain](#toolchain)).
- The project must support basic deployment with Docker and a PostgreSQL database. Start with the database service: it needs a healthcheck, a named volume for its data, and credentials/ports taken from `.env` and documented in `.env.example`. Once `backend/` has a Dockerfile, add the backend service to `docker-compose.yml` as well. Schema changes are applied only through migrations, never by editing the database by hand.
- Formatting and linting of `backend` and `frontend` are enforced by a `pre-commit` hook (lefthook), activated with `bunx lefthook install`. The hook must skip `merge` and `rebase` (`skip: [merge, rebase]`).
- Back up important data (database dumps, SQL files, and so on) into `backups/`. The directory is gitignored.
- If the user requests application metrics, use Grafana and Prometheus. Keep their configuration in a separate, opt-in `docker-compose.monitoring.yml` (not started by default) and provisioning files under `apps/monitoring/`.

### Backend Layering

Backend logic must always flow `controller -> service -> repository`.

- Controllers handle HTTP only: routing, input validation, response schemas, status codes, and error mapping. They contain no business logic and no SQL, and they never call repositories directly.
- Services hold business logic and orchestration. They depend on repository interfaces, never on concrete database implementations.
- Repositories contain SQL only and implement an interface. Concrete implementations (for example `Pg*Repository`) receive the database client through the constructor.
- Build dependencies once in the composition root and inject them; do not instantiate repositories or database clients inside services or controllers.

Base layout:

```
backend/
└── src/
    ├── modules/
    │   └── <name>/
    │       ├── <name>.controller   # HTTP boundary: validation, status codes
    │       ├── <name>.service      # business logic and orchestration
    │       └── <name>.repository   # persistence: SQL only
    └── server.ts                   # composition root
```

Exact file names and shared folders for the active stack are documented in `backend/<stack>.md`.

## Toolchain

Bun is the runtime, package manager, and test runner. The root `package.json` defines the Bun workspaces and the shared scripts. Workspaces currently cover `apps/*`; add `backend` and `frontend` to the list once those layers contain their own `package.json`. Commit the root `bun.lock`, and do not add per-layer lockfiles or use npm, yarn, or pnpm.

| Task               | Command                                        |
| ------------------ | ---------------------------------------------- |
| Install            | `bun install`                                  |
| Dev                | `bun run dev`                                  |
| Build              | `bun run build`                                |
| Test               | `bun run test`                                 |
| Test with coverage | `bun run test:coverage`                        |
| Lint               | `bun run lint` (`oxlint`)                      |
| Format             | `bun run format` (`oxfmt`), check with `bun run format:check` |
| Typecheck          | `bun run typecheck` (`tsc --noEmit`, TypeScript stacks only) |

Install git hooks once with `bunx lefthook install`; they are not active in a fresh clone until then. If a script is missing, add it to the root `package.json` instead of invoking the tool ad hoc.

## Definition of Done

A change is complete only when all of the following hold:

- The code follows the SOLID and composition rules above.
- The backend follows the `controller -> service -> repository` layering described in [Backend Layering](#backend-layering).
- `bun run lint` and `bun run format:check` pass.
- `bun run typecheck` passes for every TypeScript layer.
- `bun run test` passes, and new or changed logic is covered by tests. Run `bun run test:coverage` and report the coverage percentage; it must not be lower than the last reported value.
- Documentation is updated in the same commit: `docs/`, the relevant stack file (`backend/<stack>.md` or `frontend/<stack>.md`), and the root `README.md` when commands or setup change.
- The project has been reviewed after the change, as described in [Post-Change Analysis](#post-change-analysis).
- No secrets, dependencies, build output, coverage reports, or database dumps are committed.
- The commit message follows Conventional Commits.

## Post-Change Analysis

After every change or addition, review the project before committing:

- Re-read the diff and its call sites; remove dead code, unused imports and dependencies, leftover `TODO`/`FIXME` markers, and debug output.
- Check that references are still valid: paths in this file, links in `README.md` and `docs/`, scripts in `package.json`, variables in `.env.example`, and service names in `docker-compose.yml`.
- Confirm the change did not break adjacent behavior or contradict existing conventions, and update this file when a rule or workflow changes.
- Run the checks from [Definition of Done](#definition-of-done).
- Summarize what changed, what was verified, and any follow-up work.

## Security

- Commit `.env.example` listing every required variable; never commit `.env` or any real secret.
- Document every environment variable in `.env.example` when you add or change one.
- Ask the user before adding a runtime dependency or changing the project structure.

## Git

1. Commit completed changes, new files, and features. Do not commit code with failing tests, lint, or typecheck.
2. Use Conventional Commits, `<type>(<scope>): <subject>`.

   Examples:
   - `feat(backend): add user service`
   - `refactor(frontend): extract auth composable`
   - `fix(telegram-bot): handle empty messages`

## Bootstrap Checklist

Use this list when starting a new project from this template. Items already present in this repository are checked.

- [x] `git init`
- [x] `.gitignore`
- [ ] root `package.json` with Bun workspaces and the scripts from [Toolchain](#toolchain)
- [ ] `backend/<stack>.md` and `frontend/<stack>.md` describing each selected stack (one file per stack)
- [ ] `.env.example`
- [ ] `docker-compose.yml` with PostgreSQL
- [ ] backend service and `backend/Dockerfile` added to `docker-compose.yml` once the layer is scaffolded
- [ ] `lefthook.yml` (`pre-commit` formatting/linting, `commit-msg` Conventional Commits validation, `skip: [merge, rebase]`) and `bunx lefthook install`
- [ ] `README.md`
- [ ] `docs/ROADMAP.md`
- [ ] when metrics are requested: `docker-compose.monitoring.yml` and provisioning under `apps/monitoring/`

### Project Tree

```
ProjectName/
├── backend/
│   └── ts.md                     # backend stack description (one file per stack)
├── frontend/
│   └── vue.md                    # frontend stack description (one file per stack)
├── apps/
│   └── monitoring/
│       ├── grafana/
│       └── prometheus/
├── docs/
│   └── ROADMAP.md
├── backups/                      # gitignored
├── package.json                  # Bun workspaces and root scripts
├── bun.lock
├── docker-compose.yml
├── docker-compose.monitoring.yml # opt-in
├── lefthook.yml
├── .env.example
├── .gitignore
├── AGENTS.md
└── README.md
```
