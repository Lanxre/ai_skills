# Frontend

Stack: TypeScript + Vue 3, built with Vite.

Status: not scaffolded yet. Confirm the exact versions and state management (Pinia or other) with the user before adding dependencies.

## Tooling

- Runtime and package manager: Bun (workspace root: `../package.json`)
- Formatter: `oxfmt`
- Linter: `oxlint`
- Typecheck: `tsc --noEmit` via `vue-tsc` when Vue single-file components are present
- Tests: `bun test`, coverage via `bun test --coverage`

## Commands

Run from the repository root. See the Toolchain section of `../AGENTS.md` for the full list of scripts.
