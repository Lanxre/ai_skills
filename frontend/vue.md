# Frontend

Stack: TypeScript on Nuxt 4 (Vue 3) with Tailwind CSS 4, built and served with Bun.

## Framework and Libraries

| Area          | Package                            | Purpose                                                          |
| ------------- | ---------------------------------- | ---------------------------------------------------------------- |
| Framework     | `nuxt`                             | SSR, file-based routing, auto-imports, Nitro server              |
| UI runtime    | `vue`                              | Components, reactivity, composables                              |
| Router        | `vue-router`                       | Routing under Nuxt's file-based layer                            |
| State         | `pinia`, `@pinia/nuxt`             | Cross-page client state                                          |
| Utilities     | `@vueuse/nuxt`                     | Browser APIs, `useLocalStorage`, event helpers                   |
| Styling       | `tailwindcss`, `@tailwindcss/vite` | Utility-first CSS, CSS-first Tailwind 4 configuration            |
| UI kit        | `@kyokusu-ui/vue`                  | Shared Kyokusu components (notifications, separators, and so on) |
| Icons         | `@nuxt/icon`                       | Iconify icons (`ph:*`, `lucide:*`) with a local server bundle    |
| Theme         | `@nuxtjs/color-mode`               | Class-based light and dark themes                                |
| Charts        | `chart.js`, `vue-chartjs`          | Statistics charts                                                |
| HTTP          | Nuxt `$fetch` / `useFetch`         | API calls through `ofetch`                                       |
| Lint / format | `oxlint`, `oxfmt`                  | Linting and formatting, pinned at the workspace root             |

## Dependencies

Application (installed during the build and bundled by Nuxt):

- `nuxt`
- `vue`
- `vue-router`
- `pinia`
- `@pinia/nuxt`
- `@vueuse/nuxt`
- `@kyokusu-ui/vue`
- `@nuxt/icon`
- `@nuxtjs/color-mode`
- `tailwindcss`
- `@tailwindcss/vite`
- `chart.js`
- `vue-chartjs`

Development (tooling only):

- `oxlint` — linter
- `oxfmt` — formatter
- `browserslist`, `baseline-browser-mapping`, `caniuse-lite` — browser targets
- `@types/bun` — Bun typings for scripts and tooling
- `typescript`, `vue-tsc` — required for `nuxt typecheck`

Rules:

- Do not add `axios` or another HTTP client. Use `useApi` / `$api` from `app/composables/api/useApi.ts`.
- Do not add a second state library. Use Pinia for shared state and `useState` for SSR-safe request state.
- Do not add a component library besides `@kyokusu-ui/vue` without asking.
- Do not add another CSS framework or a CSS-in-JS runtime. Tailwind utilities plus the global stylesheet are the styling layer.
- `chart.js` is the only charting dependency; do not add a wrapper library on top of `vue-chartjs`.
- Tailwind 4 is wired through `@tailwindcss/vite` and `@import 'tailwindcss'` in `app/assets/css/global.css`. Do not add the `@nuxtjs/tailwindcss` module; if it is present in `package.json`, remove it.

## Configuration

All build-time configuration lives in `nuxt.config.ts`:

```ts
import tailwindcss from '@tailwindcss/vite'

export default defineNuxtConfig({
  modules: ['@pinia/nuxt', '@vueuse/nuxt', '@nuxtjs/color-mode', '@nuxt/icon'],

  routeRules: {
    '/api/**': {
      proxy: (process.env.NUXT_API_INTERNAL_URL || 'http://localhost:8080') + '/api/**',
    },
    '/uploads/**': {
      proxy: (process.env.NUXT_API_INTERNAL_URL || 'http://localhost:8080') + '/uploads/**',
    },
  },

  colorMode: {
    preference: 'dark',
    fallback: 'dark',
    classSuffix: '',
    storageKey: 'theme',
  },

  icon: {
    clientBundle: { scan: true, includeCustomCollections: true },
    serverBundle: 'local',
  },

  vite: {
    plugins: [tailwindcss()],
    resolve: { dedupe: ['vue'] },
  },

  build: {
    transpile: ['@kyokusu-ui/vue'],
  },

  css: ['~/assets/css/global.css'],

  runtimeConfig: {
    public: {
      apiBase: process.env.NUXT_PUBLIC_API_BASE || '',
    },
  },
})
```

- `NUXT_PUBLIC_API_BASE` — public backend origin. Use it only where the browser needs an absolute URL: uploads and avatars served from `/uploads`, OAuth redirects, and external links. Keep it empty when the frontend and API share an origin. It is inlined at build time.
- `NUXT_API_INTERNAL_URL` — target of the `/api/**` and `/uploads/**` proxy. Used during development and when Nitro runs as a server; default `http://localhost:8080`.
- `NITRO_HOST` and `PORT` — runtime binding of the Nitro server in Docker; default `0.0.0.0:3000`.
- Document every variable in `.env.example`; never commit `.env`.
- The proxy in `routeRules` is not a production reverse proxy. In production, route `/api/**` and `/uploads/**` to the backend (Caddy, nginx, or the platform router) and everything else to Nuxt.
- `runtimeConfig.public` must never contain secrets. Anything placed there is shipped to the browser.
- Tailwind 4 uses CSS-first configuration: the entry point is `app/assets/css/global.css`, which starts with `@import 'tailwindcss'` and declares the theme tokens. A `tailwind.config.ts` is not loaded by `@tailwindcss/vite`; either remove it or load it explicitly with `@config`.

## Frontend Structure

```
frontend/
├── app/
│   ├── app.vue                     # root component: layouts, global providers, notification host
│   ├── assets/
│   │   ├── css/global.css          # Tailwind entry point and global tokens
│   │   ├── fonts/                  # local fonts
│   │   └── images/                 # bundled images
│   ├── components/
│   │   ├── app/<feature>/          # feature screens and sections
│   │   ├── common/                 # header, footer, dropdowns, modals
│   │   ├── features/<Feature>/     # reusable feature widgets
│   │   └── ui/                     # design-system primitives
│   ├── composables/
│   │   ├── api/<domain>/           # use<Domain>: API access and orchestration
│   │   ├── notifications/          # notification manager
│   │   └── ui/                     # reusable UI behavior
│   ├── constants/                  # dictionaries, limits, shared URLs
│   ├── layouts/                    # default.vue, auth.vue
│   ├── middleware/                 # global and named route middleware
│   ├── pages/                      # file-based routes
│   ├── stores/                     # Pinia stores
│   ├── types/
│   │   ├── backend/                # DTOs mirroring the API
│   │   ├── frontend/               # view models and UI types
│   │   └── enums/                  # string enums shared across the UI
│   └── utils/                      # pure helpers
├── public/                         # files served as-is (favicon, robots.txt, logos)
├── nuxt.config.ts
├── tsconfig.json
├── .oxlintrc.json
├── .oxfmtrc.json
├── Dockerfile
└── vue.md                          # this file
```

Nuxt 4 keeps application code under `app/`; both `~/` and `@/` resolve to `app/`. Prefer `~/` for project code and use relative paths only inside the same directory.

## Layering

The flow is `pages -> components -> composables/api -> stores`, with `utils` and `types` available to every layer.

- **Pages** own routing, page metadata, SEO, and SSR data orchestration. They compose components and call composables; they contain no presentation details and no business rules.
- **Components** render props, emit events, and keep local UI state. They never call `$fetch`, `useApi`, or `$api` directly and never contain domain rules. A component that needs data receives it from a page or a composable.
- **API composables** (`composables/api/<domain>/use<Name>.ts`) are the only place allowed to perform HTTP calls. They build request parameters, normalize errors, expose loading state, and cache shared results with `useState` or `useAsyncData`.
- **UI composables** (`composables/ui/`) encapsulate reusable behavior: carousels, drag scrolling, image uploads, chapter editing, modal handling, reader settings. They receive refs and return state and handlers.
- **Stores** hold state that outlives a page: the authenticated user, interface settings, notifications, drafts, selected tabs. They do not perform rendering and do not import components.
- **Utils** contain pure functions only: formatting, sanitizing, sorting, URL building. No reactivity, no HTTP, no DOM access.
- **Types** are split by origin: `types/backend` mirrors the API payload exactly, `types/frontend` holds view models and UI-only shapes, `types/enums` holds string enums and their mappings.

Dependencies point downward. A composable may use a store and a utility; a store may use an API composable; a component may use a UI composable. Nothing imports a page. Avoid circular imports between stores and composables by keeping API calls in composables and state in stores.

## Pages, Layouts, and Middleware

- Routes follow the file system under `app/pages`: `index.vue`, `catalog/index.vue`, `novela/[id]/index.vue`, `profile/[id].vue`, `novela/reader/[novelaId]/[chapterId].vue`.
- `[...slug].vue` catches unknown paths and renders the not-found view; `not-found.vue` is the explicit route for `/not-found`.
- Dynamic params are validated in page middleware. A non-numeric id redirects home or throws `createError({ statusCode: 404 })` before the page renders.
- Page-level data loading uses top-level `await useAsyncData(...)` so SSR waits for the request and the payload is reused on hydration.
- `definePageMeta` declares layout, middleware, and page-specific flags. The role guard reads `minRole`:

  ```ts
  definePageMeta({
    middleware: ['auth', 'role'],
    minRole: KyokusuAppRole.MODERATOR,
  })
  ```

  Augment `PageMeta` in a `.d.ts` file instead of casting `to.meta.minRole`:

  ```ts
  declare module '#app' {
    interface PageMeta {
      minRole?: KyokusuAppRole
    }
  }
  ```

- Layouts are `default` (header, main, footer) and `auth` (minimal shell with footer). A page opts into the auth layout with `definePageMeta({ layout: 'auth' })`.
- Global middleware files are prefixed with a number to define execution order: `01.auth-init.global.ts` restores the session, `02.settings.global.ts` syncs interface settings on the server.
- Named middleware: `auth` redirects unauthenticated users to `/login`, `guest` redirects authenticated users away from login and registration, `role` enforces `minRole`.
- Never perform API calls in a component's `onMounted` when the data is needed for the first render: it breaks SSR and causes a request waterfall. Load it in the page or in a composable called from `setup`.

## Data Fetching and API Layer

`app/composables/api/useApi.ts` is the single entry point for HTTP:

```ts
import type { UseFetchOptions } from 'nuxt/app'
import { defu } from 'defu'

export function useApi<T>(url: string | (() => string), options: UseFetchOptions<T> = {}) {
  const defaults: UseFetchOptions<T> = {
    baseURL: '/api',
    headers: useRequestHeaders(['cookie']),
  }

  return useFetch(url, defu(options, defaults))
}

export async function $api<T>(url: string, options: any = {}): Promise<T> {
  const defaults = {
    baseURL: '/api',
    headers: useRequestHeaders(['cookie']),
  }

  try {
    return await $fetch<T>(url, defu(options, defaults))
  } catch (err: any) {
    const fetchError = err.data
    const errorMessage =
      fetchError?.error ||
      fetchError?.message ||
      err.statusMessage ||
      'Произошла непредвиденная ошибка'

    const error = new Error(errorMessage)
    ;(error as any).statusCode = err.response?.status
    throw error
  }
}
```

- `useApi` is SSR-aware and returns the `useFetch` result (`data`, `error`, `pending`, `refresh`). Use it for reads that should be rendered on the server.
- `$api` is imperative and throws a normalized `Error` with `statusCode`. Use it in event handlers, store actions, and composables that run after hydration.
- Use `$fetch.raw` only when response headers are required, such as cursor pagination with `x-next-cursor` and `x-has-more`. Keep it inside an API composable.
- Paths are relative to `/api`. Pass `/auth/me`, `/novela`, `/notifications`; `baseURL` adds the prefix.
- The reference contains calls written as `/api/auth/me`. They resolve to the same URL because `ofetch` skips `baseURL` when the path already starts with it. New code omits the prefix; do not rewrite existing call sites only for style.
- `useApi` must be called during setup, middleware, or another Nuxt context. In event handlers and store actions use `$api`.
- On the server, cookies are forwarded with `useRequestHeaders(['cookie'])`. On the client, requests are same-origin and the browser sends the cookie automatically. Never copy the auth cookie into JavaScript state.
- Give every `useAsyncData` a stable key that includes the resource and its parameters (`profile-${userId}`). Without a key, Nuxt derives one from the call site and the cache collides across pages.
- Fetch independent resources with `Promise.all`. Avoid sequential `await`s when the requests do not depend on each other.
- Do not set `Content-Type` for `FormData` uploads; the browser adds the boundary.
- Normalize errors in the composable, not in every component. Components receive either data or a message ready to display.

## State Management

Use Pinia setup stores for application state:

```ts
export const useAuthStore = defineStore('auth', () => {
  const user = ref<GetUserDto | null>(null)
  const isAuthChecking = ref(true)

  const isAuthenticated = computed(() => !!user.value)

  async function initAuth() {
    const { data, error } = await useApi<GetUserDto>('/auth/me')
    user.value = !error.value && data.value ? data.value : null
    isAuthChecking.value = false
  }

  return { user, isAuthChecking, isAuthenticated, initAuth }
})
```

- File name: `app/stores/<name>.ts`; store name: `use<Name>Store`; store id: `<name>`.
- Setup stores are the default. The options form is acceptable only for state with persistence or many actions, such as `chapterDraft`.
- Stores expose state and actions, not raw setters that break invariants. Mutate state through actions; do not assign `authStore.user` from a component.
- Use `storeToRefs` when destructuring reactive state.
- `useState('<key>', () => initial)` is for SSR-safe state shared between middleware, pages, and composables. Keys are global strings; namespace them (`notifications-list`, `profile-fetched-user`) and never create module-level `ref`s that leak between SSR requests.
- Persist to `localStorage` only on the client. Guard with `import.meta.client`; read storage lazily, not at module scope, so SSR does not crash or leak data between users.
- Do not mirror server data in a store just to share it. Keep fetched resources in the composable that owns them; use a store only when the state must survive navigation or drive global UI.

## Components

- Component directories map to responsibility: `ui/` for generic primitives (`Table`, `LightBoxImage`, `RangeSlider`), `common/` for application chrome (`HeaderApp`, `FooterApp`, `SearchModal`), `features/` for reusable feature widgets (`PageLayout`, `TagSelector`, `YearSelector`), `app/<feature>/` for screens and sections of one feature.
- Use PascalCase file names with at least two words where the name would otherwise collide with an HTML element or a UI-kit component.
- Always use `<script setup lang="ts">`. Declare props with a typed interface and `withDefaults`, emits with `defineEmits<{ ... }>()`.
- Import components explicitly from `@/components/...`. Nuxt auto-import would produce path-based names (`AppAuthLoginPage`, `UiTable`), which hides the origin; explicit imports keep the dependency graph readable.
- Props go down, events go up. A component that needs to change page data emits an event or calls a composable passed to it, rather than importing the page's composable directly.
- Extract reusable logic into a composable instead of duplicating it in components. Extract repeated markup into a component under `ui/` or `features/`.
- Wrap browser-only components in `<ClientOnly>` and guard direct `window`, `document`, and `localStorage` access with `import.meta.client`.
- Use `<Icon name="ph:..."/>` from `@nuxt/icon`; do not add raw SVG markup unless it is a brand asset.
- Use semantic elements and accessible names: `<button>` for actions, `<a>`/`<NuxtLink>` for navigation, labels bound to inputs, and `aria-*` attributes on custom widgets.
- Keep templates declarative. Move branching and formatting into computed properties or utilities.

## Types and Constants

- `app/types/backend/*.ts` mirrors API DTOs field for field, including `snake_case` names. Do not rename fields in the type or transform responses in an ad hoc way; add a mapping function in `utils` when the view needs a different shape.
- `app/types/frontend/*.ts` holds view models, form state, and UI-only types. Never send a frontend type to the API without mapping it to the backend contract.
- `app/types/enums/*.ts` holds string enums for roles, tabs, themes, genders, social networks, and statuses. Do not compare against string literals scattered through the code.
- Role ordering is defined once:

  ```ts
  export const ROLE_WEIGHTS: Record<KyokusuAppRole, number> = {
    [KyokusuAppRole.USER]: 10,
    [KyokusuAppRole.PUBLISHER]: 30,
    [KyokusuAppRole.MODERATOR]: 50,
    [KyokusuAppRole.ADMIN]: 100,
  }
  ```

- Shared dictionaries and limits live in `app/constants`. Use `as const` for literal lists so unions can be derived from them.
- Import types with `import type`; do not import a type as a value.
- Prefer `unknown` plus a narrowing check over `any` at API boundaries. The reference uses `any` in error handling; new code should introduce small typed helpers instead.

## Styles

> **Reserved section.** Styling conventions are intentionally not defined yet. Fill this section in before building the first screen, and keep the decisions in one place instead of scattering them across components.

Document at least:

- **Design tokens** — which CSS custom properties exist, their names, and where they are declared (the global stylesheet is `app/assets/css/global.css`).
- **Typography** — font families, sizes, weights, line heights, and the use case for each level.
- **Colors** — the palette and its semantic roles: background, surface, border, text, muted text, accent, success, warning, danger. Include contrast requirements.
- **Spacing** — the spacing scale, container widths, page padding, and gaps between sections.
- **Dark mode** — the class strategy, the default theme, how tokens switch between themes, and what must never be hard-coded.
- **Breakpoints** — mobile, tablet, desktop, and wide-screen thresholds, and the layout behavior at each one.
- **Component conventions** — variants, sizes, states (default, hover, focus, disabled, loading), radii, borders, shadows, and which primitives belong in `app/components/ui`.

Until this section is complete, do not introduce a parallel token system, a second global stylesheet, or component-level overrides that depend on hard-coded colors.

## SEO and Metadata

- Set global defaults in `nuxt.config.ts` under `app.head`: charset, viewport, title, favicon, and `htmlAttrs: { lang: 'ru' }`.
- Set page metadata with `useSeoMeta`, not `useHead`, for titles, descriptions, Open Graph, and Twitter cards.
- Use functions for values that depend on fetched data:

  ```ts
  useSeoMeta({
    title: () => (profileData.value?.name ? `${profileData.value.name} - Профиль` : 'Профиль'),
    ogTitle: () => profileData.value?.name,
    ogImage: () => profileData.value?.picture,
  })
  ```

- Title pattern: `<Page> - KyokusuLib`. Pages without a specific title fall back to the global title.
- Use `useHead` only for tags that are not covered by `useSeoMeta`, such as preloads or JSON-LD.
- Keep `public/robots.txt` current. Add a sitemap module only with the user's approval.

## Authentication and Authorization

- Authentication uses the httpOnly cookie `KYOKUSU_API_TOKEN`. The browser never reads it; JavaScript must not try.
- `01.auth-init.global.ts` runs on every route:

  ```ts
  export default defineNuxtRouteMiddleware(async () => {
    const authStore = useAuthStore()
    const token = useCookie('KYOKUSU_API_TOKEN')

    if (import.meta.server) {
      if (token.value && !authStore.isAuthenticated) {
        await authStore.initAuth()
        if (!authStore.isAuthenticated) token.value = null
      }
      return
    }

    const initialized = useState('auth-initialized', () => false)
    if (initialized.value) return
    initialized.value = true

    if (!authStore.isAuthenticated) await authStore.initAuth()
  })
  ```

- On the server, `useCookie` can read the httpOnly cookie and `useApi` forwards it to the backend. On the client, the session is restored from the SSR payload; `useState('auth-initialized')` prevents a second `/auth/me` request after hydration.
- Login posts credentials, stores the returned user in the auth store, and navigates to the profile. Logout posts to `/auth/logout`, clears the store, and redirects to `/login`.
- OAuth login is a full-page redirect to `${apiBase}/api/auth/<provider>/login`; do not open it in a popup and do not try to read the callback token.
- Route protection:
  - `auth` — requires an authenticated user;
  - `guest` — only for anonymous users;
  - `role` — requires `minRole`, compared through `ROLE_WEIGHTS`.
- UI-level checks (`v-if="hasPermission(...)"`, role comparisons) hide controls; they are not security. The backend enforces every permission.
- Clear user-specific `useState`, disconnect streams, and reset stores on logout so the next user never sees cached data.

## Notifications and Real-time

- Notifications arrive over Server-Sent Events: `new EventSource('/api/notifications/stream', { withCredentials: true })`.
- `EventSource` does not go through `useApi` or `$fetch`, so it ignores `baseURL`. Pass the same-origin absolute path, including `/api`.
- Connect after authentication, disconnect on logout, and reconnect with exponential backoff: `Math.min(1000 * 2 ** attempts, 30000)`.
- Do not open a stream during SSR. Guard with `import.meta.server` and start it from a client-only lifecycle.
- Keep the stream in `useState` so navigation does not create duplicates, and close it when the owning component unmounts if the stream is page-scoped.
- Parse event payloads defensively: a malformed message must not break the stream.
- Toast notifications go through `useNotificationStore().notify({ type, title, content })`. The store deduplicates identical messages for 3 seconds and dismisses them after 5 seconds by default.
- Render the notification host once in `app.vue` inside `<ClientOnly>`.
- Cursor-based feeds use the shared pagination headers `x-next-cursor` and `x-has-more`; keep the cursor in composable state, not in the URL unless the page must be shareable.

## Errors and Loading

- `$api` throws a normalized `Error` with `statusCode`; catch it at the composable boundary and return a safe value or rethrow with context.
- `useApi` returns `error` instead of throwing. Check `error.value` before using `data.value`.
- Fatal page errors use `createError({ statusCode: 404 })` (or `403`) and render through the app error page. Add `app/error.vue` when the first fatal page is introduced.
- Recoverable errors surface as a toast; do not replace the whole page for a failed mutation.
- Never leave an empty `catch {}` in new code. Log with context in development and show a generic message to the user.
- Every asynchronous action exposes a loading flag and disables its trigger while pending. Do not fire duplicate mutations.
- Prefer skeletons for content-shaped loading and a spinner for short actions. Reserve space for the final layout to avoid shifts.

## TypeScript and Tooling

- `tsconfig.json` only references the generated Nuxt configs. `nuxt prepare` (run by `postinstall`) creates `.nuxt/tsconfig.*.json`; never edit generated files.
- Keep strict typing. Do not disable `strict` or add blanket `// @ts-ignore`; use `unknown` and narrow.
- `nuxt typecheck` requires `typescript` and `vue-tsc`. Add the script to `package.json`:

  ```json
  {
    "scripts": {
      "typecheck": "nuxt typecheck",
      "check": "bun run typecheck && bun run lint && bun run format:check"
    }
  }
  ```

- `oxlint` is configured in `.oxlintrc.json`: `correctness` is an error, `suspicious` and `perf` are warnings. Do not silence a rule globally to fix one file.
- `oxfmt` is configured in `.oxfmtrc.json`: print width 100, single quotes, no semicolons, trailing commas, 2-space indentation, LF line endings.
- Tests: the reference frontend has no runner configured. Agree on one with the user before adding tests; the usual choice for Nuxt is Vitest with `@nuxt/test-utils`. Once configured, wire `bun run test` and `bun run test:coverage` at the workspace root.
- The root `lefthook.yml` runs formatting and linting before commit; install it once with `bunx lefthook install`.
- Run `bun run check` before committing. It must pass typecheck, lint, and formatting.

## Docker

- Multi-stage build on `oven/bun:1`:
  - `deps` — copy `package.json` and `bun.lock`, run `bun install --frozen-lockfile`;
  - `build` — copy dependencies and source, accept `NUXT_PUBLIC_API_BASE` and `NUXT_API_INTERNAL_URL` as build args, run `bun run build`;
  - `runner` — copy `.output`, run `bun .output/server/index.mjs`.
- `NUXT_PUBLIC_API_BASE` and `NUXT_API_INTERNAL_URL` are inlined during `nuxt build`; passing them only at runtime does not change the client bundle. Rebuild the image when they change.
- `EXPOSE 3000`; Nitro reads `NITRO_HOST` and `PORT` from the environment.
- `docker-compose.yml` builds the service from `./frontend`, passes the two build args, sets `NITRO_HOST=0.0.0.0` and `PORT=3000`, and depends on the backend.
- The container needs a reverse proxy in front of it: `/api/**` and `/uploads/**` go to the backend, everything else to Nuxt. The development proxy in `routeRules` is not used by the static production proxy.
- Keep `.dockerignore` aligned with `.gitignore`: exclude `node_modules`, `.nuxt`, `.output`, coverage, and `.env`.
- Run the final image as a non-root user in production.

## Commands

Run from `frontend/` with Bun:

```bash
bun install           # install workspace dependencies
bun run dev           # development server with HMR on http://localhost:3000
bun run build         # production build into .output
bun run preview       # preview the production build
bun run generate      # static generation, when the target is static hosting
bun run lint          # oxlint
bun run lint:fix      # oxlint --fix
bun run format        # oxfmt --write
bun run format:check  # oxfmt --check
bun run typecheck     # nuxt typecheck
bun run check         # typecheck + lint + format:check
bun run fix           # lint:fix + format
```

See the Toolchain section of `../AGENTS.md` for the root scripts that wrap these commands.
