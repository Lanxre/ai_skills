# App Development Guidelines

1. Follow the SOLID principles. Concretely: keep domain logic out of controllers and UI components, depend on interfaces rather than implementations, and inject dependencies instead of constructing them inline.
2. Prefer composition (aggregation) and delegation over inheritance. Do not build deep inheritance chains.

## Architecture

The application is split into two main layers, `backend` and `frontend`, plus standalone services.

- Before writing code, read the stack description in `backend/README.md` and `frontend/README.md`. If the relevant file does not describe the stack you need, stop and ask the user.
- Standalone runnable services (for example, a Telegram bot) go into `apps/`. Only self-contained applications belong there, and each has its own `README.md`.
- Project documentation and specifications go into `docs/`. The only exceptions are `README.md` and `AGENTS.md` in the repository root, and `backend/README.md` / `frontend/README.md` for stack descriptions.
- Do not add new top-level directories without asking the user.
- The project is a single Bun workspace rooted at the repository root; all commands are run from there (see [Toolchain](#toolchain)).
- The project must support basic deployment with Docker: the selected backend stack plus a PostgreSQL database. The database service needs a healthcheck, a named volume for its data, and credentials/ports taken from `.env` and documented in `.env.example`. Schema changes are applied only through migrations, never by editing the database by hand.
- Formatting and linting of `backend` and `frontend` are enforced by a `pre-commit` hook (lefthook). It must skip `merge` and `rebase` (`skip: [merge, rebase]`).
- Back up important data (database dumps, SQL files, and so on) into `backups/`. The directory is gitignored.
- If the user requests application metrics, use Grafana and Prometheus. Keep their configuration in a separate, opt-in `docker-compose.monitoring.yml` (not started by default) and provisioning files under `apps/monitoring/`.

## Toolchain

Bun is the runtime, package manager, and test runner. The root `package.json` defines the workspaces (`backend`, `frontend`, `apps/*`) and the shared scripts. Do not add per-layer lockfiles and do not use npm, yarn, or pnpm.

| Task               | Command                                        |
| ------------------ | ---------------------------------------------- |
| Install            | `bun install`                                  |
| Dev                | `bun run dev`                                  |
| Build              | `bun run build`                                |
| Test               | `bun test`                                     |
| Test with coverage | `bun test --coverage`                          |
| Lint               | `bun run lint` (`oxlint`)                      |
| Format             | `bun run format` (`oxfmt`), check with `bun run format:check` |
| Typecheck          | `bun run typecheck` (`tsc --noEmit`, TypeScript stacks only) |

If a script is missing, add it to the root `package.json` instead of invoking the tool ad hoc.

## Definition of Done

A change is complete only when all of the following hold:

- The code follows the SOLID and composition rules above.
- `bun run lint` and `bun run format:check` pass.
- `bun run typecheck` passes for every TypeScript layer.
- `bun test` passes, and new or changed logic is covered by tests. Run `bun test --coverage` and report the coverage; do not lower the existing level.
- Documentation is updated in the same commit: `docs/`, the relevant layer `README.md`, and the root `README.md` when commands or setup change.
- No secrets, dependencies, build output, coverage reports, or database dumps are committed.
- The commit message follows Conventional Commits.

## Security

- Commit `.env.example` listing every required variable; never commit `.env` or any real secret.
- Document every environment variable in `.env.example` when you add or change one.
- Ask the user before adding a runtime dependency or changing the project structure.

## Git

1. Initialize a new project with `git init`.
2. Commit completed changes, new files, and features. Do not commit code with failing tests, lint, or typecheck.
3. Use Conventional Commits, `<type>(<scope>): <subject>`.

   Examples:
   - `feat(backend): add user service`
   - `refactor(frontend): extract auth composable`
   - `fix(telegram-bot): handle empty messages`

## Bootstrap Checklist

The tree below is the target layout. A new project is ready only when every item is checked.

- [ ] `git init`
- [ ] root `package.json` with Bun workspaces and the scripts from [Toolchain](#toolchain)
- [ ] `.gitignore`
- [ ] `backend/README.md` and `frontend/README.md` describing the selected stacks
- [ ] `docker-compose.yml` with the backend stack and PostgreSQL
- [ ] `lefthook.yml`: `pre-commit` formatting/linting, `commit-msg` Conventional Commits validation, `skip: [merge, rebase]`
- [ ] `.env.example`
- [ ] `README.md`
- [ ] `docs/ROADMAP.md`

### Project Tree

```
ProjectName/
├── backend/
│   └── README.md                 # backend stack description
├── frontend/
│   └── README.md                 # frontend stack description
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
