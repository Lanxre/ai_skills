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
| Icons         | `@nuxt/icon`                       | Iconify icons, `ph` first, with a local server bundle            |
| Theme         | `@nuxtjs/color-mode`               | Class-based light and dark themes                                |
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
- `@iconify-json/ph` — primary icon collection
- `@iconify-json/lucide` — fallback only
- `@nuxtjs/color-mode`
- `tailwindcss`
- `@tailwindcss/vite`

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

These rules are the visual contract for the frontend. Concrete values live as tokens in `app/assets/css/global.css` and are consumed through Tailwind utilities or `var(--...)`; do not hard-code a value that has a token. Three terms are used throughout:

- **Direction** — the deliberate visual direction chosen for the product.
- **World** — the product's subject matter and the visual language borrowed from it.
- **Moments** — the few focal places where the product itself is shown: a chart, a rendered object, a real image.

Define the token set before styling a screen: the neutral ramp, the three text roles, the accent, semantic colors, the spacing scale, the type scale, radii, shadows, breakpoints, and motion durations. Add the values to `app/assets/css/global.css`; components consume them and never duplicate them.

A screen is not done until it passes every section below.

### Spacing

- Everything sits on a 4px grid. Margins, paddings, gaps, and fixed heights are multiples of 4, so 28 or 104 are fine even though they are not in the token list. A 2px step is fine for nudges inside small controls, and an odd value that exactly centers something stays.
- Same thing, same size. All rows in a list share one height and one gap; all cards in a row share one padding.
- Space shows grouping. Space inside a group is smaller than space between groups, so a list reads as one block and sections read as separate ones.
- Things line up. Icons sit in one column, labels start on one line, numbers align right in one column. Nothing is nudged off the line with a stray translate or odd margin.
- Padding follows density. Compact panels and sidebars sit around 16, cards at 20 to 24, page edges at 24 to 32.
- Navigation is the densest part of an app. Rows are 32 to 36 tall with 2 to 4 between them, icons in one column, labels on one line. A section label sits closer to its own items than to the section above it, like 24 above and 8 below. Check the sidebar after the pass; it is where spacing breaks most often.
- Remove one-off values that break the rhythm, like `mt-[13px]` or `py-[11px]`, and replace them with steps from the scale.

### Color

- Build from one neutral ramp, tinted toward the world of the direction. Pick graphite, warm, or ink, or tint your own ramp the same way. Background, raised surfaces, borders, and text all come from that ramp.
- In apps and dashboards, backgrounds, surfaces, borders, and text stay neutral: the graphite ramp or something just as grey. People look at them for hours, and the content has to carry the color. A screen whose background reads as orange, brown, blue, or green has gone too far. The world shows in the accent, the charts, and the moments, not in the surfaces. Landing pages can tint their surfaces further toward the world.
- Three text roles: primary, secondary, and muted. Not ten slightly different lavenders.
- In the chrome, one accent, used rarely. It marks the primary action, the selection, or the one number that matters.
- The moments from the direction are the exception. Inside them color can be rich: several hues in a chart, a rendered object with real materials, an image with its own light, as long as it stays inside the world and each color means something.
- If the brand has a real color, it becomes the accent and leaves the surfaces. If there is no real brand color, take the accent from the world.
- An accent with text on it keeps at least 4.5:1 contrast. If it does not, darken it or put dark text on it. An accent used as text or ink on a light surface needs the same contrast, so use a darker shade there.
- On dark themes, the primary button can be an off-white from the ramp with dark text, and the accent can live in a smaller place like the selection or a key figure. This often looks more deliberate than a colored button.
- Semantic colors only carry meaning, like green for a good change and red for an error, and they are toned down to sit inside the palette.
- Remove hue gradients on surfaces, gradient text, colored glows and colored shadows, radial color washes and blobs behind content, and neon borders.
- Borders are quiet hairlines: the text color of the ramp at 6 to 10% opacity.
- On dark themes, depth comes from surface steps and hairlines, not from shadows. A window or object floating over an image may keep a soft neutral shadow. On light themes, shadows are neutral, soft, and small.
- A gradient is allowed only as a small tone shift inside one neutral, like an active row going from `#2b2b2b` to `#1f1f1f`. Never from one hue to another.

### Icons

- Every emoji used as an icon or as decoration in a label becomes a line icon. Use `ph` first. Reach for `lucide` only when `ph` has no suitable icon, and never mix sets on one surface.
- One size and one stroke everywhere: 16 or 18px with a 1.5 stroke.
- Icons take the color of their text role, usually secondary or muted. They never get their own colors.
- The icon box has the same width in every row and the gap to the label is fixed at 8 or 10, so labels align.
- Drop icons that add nothing, like an emoji in every card title or before every section label. Categories that need color get a small status dot.

### Copy

- Headlines say what the product does for whom, in plain words, so they could not fit another product. Use the facts already on the page and do not invent new ones.
- No exclamation marks in interface text, and never `!!`.
- No greeting headlines like `Welcome back, Damir! 👋`. The title names the context, such as the page name or the period, for example `October 2026`.
- Short labels. `Search`, not `Search anything...`.
- A change says what it compares to, like `+12% vs last month` or `3 fewer vs last month`.
- Headlines, subheads, labels, buttons, and captions end without a period. Only paragraphs of several sentences keep their periods.
- No hype words like supercharge, seamless, unlock, magic, revolutionize, or game-changing, and no `AI-powered` on everything.
- Claims that cannot be checked, like `Loved by 10,000+ teams` or `10x faster`, become quiet and specific if the user confirms they are real, and are removed otherwise. Ask the user and record the answer in the change summary.
- Mock data looks real, with plausible numbers and names, never `[X]` or `123,456`.
- Badges like `PRO`, `New ✨`, or `Beta 🚀` go away unless the product really needs them, and then they are quiet text.

### Type

- Choose type for the character of the direction, from the families in the tokens. Keep the project's font when it is part of a real brand. A default like Inter that nobody chose may be swapped when another family fits the character better. One family for the interface, plus at most one partner from any row of the tokens: a mono for figures and meta, or a display face for large headlines.
- A few sizes from a scale and two weights, 400 and 500, with 600 at most for rare emphasis. No 700 to 900 headings, no gradient text.
- Hierarchy comes from size and color, not from weight.
- Sentence case everywhere. Labels like `NAVIGATION` or `PAID THIS MONTH` with wide tracking become `Navigation` and `Paid this month`.
- Small text never gets negative letter spacing. Readability on small text means bigger, a bit bolder, or slightly looser. Tight tracking is only for large display text.
- Numbers use `tabular-nums` and one format across the screen. If one figure is `$48,210`, the next is not `$124,860.40`.
- No italic for emphasis.

### Layout

- Keep the sections and their order. Inside a section, alignment and proportions may change when that makes the screen clearer.
- Apps and dashboards keep real app proportions. The shell fills the viewport, a sidebar is 240 to 280 wide, content is fluid and usually capped around 1440 to 1600, and the page is as long as its content, with more sections below the first screen. Never shrink an app into a narrow centered column. If the original was drawn as a small window on a backdrop, keep it visibly a window, with its frame and backdrop, and still give it desktop proportions inside.
- On landing pages, the strongest hero is a scene. One generated image fills the hero from edge to edge, its subject sits on one side, and the headline sits in the calm empty part of the same image, with the product view rising from the bottom edge.
- Without a scene, use an asymmetric split: text on one side and the visual given at least half the width, or text over a large product view below it. Everything stacked and centered is the template default; keep it only for a short hero with no visual.
- In a scene hero, the text block sits in the upper part, in the space between the navigation and the top of the product view, at its optical center, a little above the true middle.
- In a split, center the text block vertically against the visual, so neither side leaves a big empty corner. If the visual is much taller, shorten it instead of pushing the text down. Floating cards and devices next to the text are hard to balance; prefer one solid visual. For stacked heroes, use the spacing token that sets the distance from the navigation to the headline; a split hero is centered instead.
- Long headlines align left. Centered text works for one or two short lines.
- One primary action and at most one secondary action next to it. Anything else moves out of the hero.
- Navigation stays quiet: logo, the existing links, one button. Log in is a quiet text link next to it, not a second button. A floating pill header is not required. On small screens the links collapse into one menu button.
- Compose the first screen as a whole. The hero fills the viewport height without a dead empty band. If the content is shorter, center it or let the product view start low and run past the bottom edge, so the page invites a scroll.
- On small screens everything stacks in reading order, the headline drops to the mobile size from the tokens, and the visual comes after the actions. A dense preview gets simpler on mobile, with fewer rows and columns, instead of being zoomed down. No text in it goes below 11px.

### Details

- Nested radius: an inner radius equals the outer radius minus the padding between them, never less than 4. Controls use 8 to 10, cards 12 to 16, panels 14 to 20. Not everything is a pill.
- Counts and meta become quiet text, often mono and muted, instead of glowing pills.
- Keyboard shortcuts become small keycaps.
- Charts sit on a muted grid with muted mono labels. No rainbow bars inside one series, no glowing bars, no uppercase day labels. If the chart is placeholder noise, replace it with data that tells something real.
- Charts have a hierarchy, like text. One main chart is clearly the biggest thing on the screen, drawn with care, such as smooth lines or a mirrored comparison. Anything else that shows data is smaller and different in kind, like a ranked list with thin bars or a short table, never a second chart of the same size.
- One accent per chart. The thing that matters, such as the current month, the line being tracked, or the plan that grew, gets the accent, and everything else sits in neutral steps of the ramp. Several hues only when the chart really has to tell series apart, and even then one leads and the rest are quiet.
- Stat cards are numbers: a label, a big figure, and a change with what it compares to. No sparklines, tallies, mini bars, or icons added to fill them; they read as decoration. A card can hold a real-looking object when it is the very thing its number counts, like a paid receipt next to paid revenue or bank cards next to spend. Use as many as make sense for that dashboard, as long as each one reads on its own and none is a mini chart.
- Vary the sizes. Cards of equal width side by side, each with the same kind of content, read as a template. Give the grid a rhythm, like three or four compact stats in a row above one wide chart, or a wide chart next to a narrow list.
- Pick the chart type for the data. A trend is a line, a comparison is bars, a share is one stacked bar or a list with bars, and a single goal is a progress bar.
- The active state is a small surface step with a hairline highlight, not a gradient with a glow.
- Hover and focus states exist and stay inside the palette. Keyboard users get a visible focus ring.
- The logo is the product name in the main family, plus a simple mark if there is one. No gradient wordmark and no sparkle tile.
- The signature detail from the direction appears where it belongs and nowhere else.

### Visuals

- Decorative filler goes: glossy blobs, orbs, nebulas, floating cubes, glowing grids, holographic brains, and robots say nothing about the product.
- A landing hero gets a visual that shows something true.
- Dashboards and app screens almost never need generated images. Their moments come from the product itself: a chart drawn with care, figures set beautifully, real content like product photos or avatars, a status that feels alive. A physical thing the product deals with, like a payment card in a finance app, is usually better built in code with real materials, a soft gradient, a chip, and a number.
- Generate images for an app only for empty states, onboarding, and marketing moments, or when the user asks for it.
- With an image generation tool, the first choice for a landing hero is a scene from the world in the direction, generated wide enough to sit behind the whole hero, with calm empty space where the text goes. It carries the color and the mood, and the chrome stays quiet over it.
- Under the scene, or as the visual when there is no tool, show the product. Build a preview in code of what the page already promises, with the project's components or careful HTML and plausible data, such as the main screen, one key card, or the result the product produces. Showing what the copy already claims is not inventing a feature. In a scene hero it rises from the bottom edge and runs past it.
- The product view carries a moment, not just a plain card: a rendered object inside it, like a payment card on a figure, a rich chart, or a small object from the world. These are kinds of objects, so using one that fits is fine.
- Before this pass, review the reference material for the direction. Take qualities from it, such as composition, light, restraint, and how the moment is placed; never its subject or layout.
- Rendered objects and scenes come from an image generation tool when one is available. If not, build what you can in code, write the prompts for the rest, and leave empty slots of the right size.

### Motion

- Remove motion that loops for no reason, like `animate-pulse` or `animate-ping` on things that are not live, shimmer on headings, breathing glows, floating elements, and parallax.
- Remove `transition-all`, bouncy springs, `hover:scale` on cards, and anything that moves more than a few pixels on hover.
- Keep motion that explains something. A hover is a quick color or surface change, a press is a tiny scale down, a menu fades and rises a few pixels from where it opens, and a changed number tweens.
- Transition color, background, border, opacity, and transform, and name them, like `transition-colors` or `transition-[transform,opacity]` in Tailwind, instead of a catch-all. Never animate width, height, top, left, margins, or large blurs; they stutter on weak devices.
- One easing curve and a few durations across the screen, from the tokens. Short for feedback, a little longer for things that enter.
- At most one entrance on load: a short fade and rise for the main content with a small stagger. Not every card flying in from a different side.
- Respect `prefers-reduced-motion`: drop non-essential transitions and entrances when the user asks for reduced motion.

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
