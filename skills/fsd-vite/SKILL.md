---
name: fsd-vite
description: Apply Feature-Sliced Design (FSD) v2.1 architecture to React + Vite projects (including TanStack Router/Query setups). Use this skill ONLY when the user explicitly mentions FSD-related topics such as: "FSD", "feature-sliced", layer names (app, pages, widgets, features, entities, shared), slices, segments, import rule violations, or cross-imports; OR asks a placement question ("where should I put X", "which layer does Y go in", "can I import X from Y"); OR requests scaffolding, refactoring, or architecture review of a Vite/React project structure. Do NOT trigger for general React/Vite questions unrelated to project structure or code placement.
---

# Feature-Sliced Design v2.1 — React + Vite

FSD is a layered architecture for frontend apps. This skill is the React+Vite variant. Core FSD rules are universal; Vite is the *easy* case for FSD because there is no framework-imposed routing folder fighting the structure (unlike Next.js). You own routing explicitly (e.g. TanStack Router, React Router).

## The one rule that matters most: the import rule

A module in a slice can import only from layers **strictly below** it. Same-layer and upward imports are forbidden.

| Layer | Can import from |
|-------|----------------|
| `app` | pages, widgets, features, entities, shared |
| `pages` | widgets, features, entities, shared |
| `widgets` | features, entities, shared |
| `features` | entities, shared |
| `entities` | shared only — plus other entities via `@x` (see below) |
| `shared` | nothing above it |

**Violation signals:**
- feature imports another feature → lift composition to a widget or page
- entity imports another entity directly → use `@x` notation
- anything imports from a layer above itself → restructure, no exceptions

## Layers (top to bottom)

1. **app** — app initialization: providers (QueryClientProvider, RouterProvider, theme), global styles, entrypoint wiring.
2. **pages** — full pages. Each page is a slice. Pages cannot import other pages.
3. **widgets** — large self-contained UI/logic blocks delivering a complete use case (header, comments section). Use when a reusable block needs layers above shared.
4. **features** — *reused* user interactions with business value (`auth/login`, `add-to-cart`). Not everything is a feature.
5. **entities** — business entities (`user`, `product`, `order`): domain reads + domain UI primitives.
6. **shared** — business-agnostic reusable code: UI kit, API client, config, lib, router primitives. No business logic.

`processes` is **deprecated** — do not create it.

`app` and `shared` have **no slices** — straight to segments. `pages`, `widgets`, `features`, `entities` MUST have slices, with segments inside slices.

## Slices and segments

- **Slices** partition a layer by business domain. Slices on the same layer cannot import each other.
- **Segments** partition a slice by technical purpose:
  - `ui` — components, styles, formatters
  - `api` — backend interactions, request functions, mappers
  - `model` — schemas, stores, business logic, types
  - `lib` — internal helper code for this slice
  - `config` — feature flags, constants
- Segment names describe **purpose (why), not essence (what)**. Never `components/`, `hooks/`, `utils/`.

## Public API (mandatory)

Every slice exposes a contract via `index.ts`. Outside code imports only from the slice root.

```ts
// features/auth/index.ts
export { LoginForm } from "./ui/LoginForm";
export { useAuth } from "./model/useAuth";
```

For **shared**, define a public API per segment (`shared/ui`, `shared/api`) rather than one giant index.

## Pages First (the v2.1 mental model)

Default to placing code **where it is used**. Start in `pages`/`widgets`. Extract to `features`/`entities` ONLY on real reuse — don't predict the future.

- Form used on one page → that page's `ui` segment.
- Same interaction appears in **at least 2 distinct slices** → extract to `features`. One confirmed future use case (e.g. same feature needed in an upcoming page) also qualifies. Predicted reuse without evidence does not.
- Extraction is premature if the code is used in **only one place and no confirmed reuse exists**. If a slice is imported in exactly one location — it belongs in that location's layer instead.

Caveat for monorepo / shared-backend work: if a slice (e.g. `auth`) is genuinely shared across packages, extracting early is justified — "reuse" there is cross-package, not in-app. Judge reuse at the right scope.

## `@x` notation — the ONLY way entities cross-import

```
entities/
  user/
    @x/order.ts       ← public API exposed specifically TO the order entity
    index.ts          ← regular public API
    model/user.ts
  order/
    model/order.ts
```

```ts
// entities/user/@x/order.ts
export type { User } from "../model/user";

// entities/order/model/order.ts
import type { User } from "@/entities/user/@x/order";
```

Read `A/@x/B` as "A crossed with B". Entities layer only, minimal use, last resort. Heavy `@x` usage means entities are split too granularly — merge them.

## Vite project structure & path aliases

Standard layout (no routing-folder conflict):

```
project-root/
├── index.html
├── vite.config.ts
├── tsconfig.json
└── src/
    ├── app/
    │   ├── providers/
    │   ├── styles/
    │   └── index.tsx        ← root: mounts providers + router
    ├── pages/
    ├── widgets/
    ├── features/
    ├── entities/
    └── shared/
```

Set up the `@` alias to `src` so imports read `@/features/auth` etc.

```ts
// vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import path from "node:path";

export default defineConfig({
  plugins: [react()],
  resolve: { alias: { "@": path.resolve(__dirname, "./src") } },
});
```

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

## Routing (TanStack Router / React Router)

Routing config lives in the **app** layer (it composes pages, which is allowed top-down). Route definitions import page components from `pages` and below — never from deep internals.

- Prefer **direct component imports** in route definitions over `lazy`-everything by default; introduce lazy/code-splitting deliberately where it pays off, not reflexively.
- Keep route files as thin composition: a route maps a path to a page slice's public API.

```tsx
// app/router.tsx (conceptual)
import { createRoute } from "@tanstack/react-router";
import { HomePage } from "@/pages/home";
// route → page slice public API, nothing deeper
```

Shared routing primitives (route constants, guards) → `shared/router` or `shared/config`.

## API requests

- Shared, reusable requests → `shared/api` (`client.ts` wrapper + `endpoints/` grouped by endpoint, re-exported from `index.ts`).
- Request used by exactly one slice → that slice's `api` segment; needn't be in the slice's public API.
- Don't prematurely dump backend response types into `entities` — transform in `shared/api` or the slice `api` segment; keep entities frontend-focused.
- OpenAPI backend → generate into `shared/api/openapi` with a regeneration README.

## TanStack Query

- Centralize cache keys, query/mutation option factories, and shared API data types in `shared` so invalidation stays consistent.
- Co-locate query/mutation hooks with the slice that owns the data — often `entities/<x>/api` for reads, or the consuming `feature` for mutations.
- `QueryClient` + `QueryClientProvider` belong in the **app** layer.

## Forms & validation

- Validation schemas → `model` segment of the owning slice (page/feature), exposed to `ui`.
- With `@hookform/resolvers`, use **Zod v3**, not Zod v4 — v4 is not compatible with the resolver. Pin accordingly.

## Auth & token storage

- Login/register → one `pages/login` slice (group both forms). A login dialog usable anywhere → a widget.
- Token storage: SPA without a server side usually can't rely on httpOnly cookies the same way an SSR app can. Common FSD-compatible options:
  - Store in `shared` alongside the API client (enables a refresh middleware: on 401, refresh + retry).
  - Store in an `entities/user` (current-user / "viewer") model store, exposing the token to the API client via context/global store or an injection-on-change subscription.
  - Do NOT store app-wide tokens in pages/widgets.
- If access token + refresh are handled differently (e.g. accessToken in memory, a refresh identifier in localStorage), keep the storage/refresh logic in one place — `shared/api` or a dedicated `shared/auth` segment — so it doesn't dilute the api segment.

## Decision checklist (where does this code go?)

1. Business-agnostic infra (button, fetch wrapper, date util)? → `shared`.
2. Business entity's reused data/UI primitive? → `entities/<name>`.
3. Reused user interaction with business value? → `features/<name>`.
4. Big reusable UI block needing layers above shared? → `widgets/<name>`.
5. Specific to one page, not yet reused? → `pages/<name>` (Pages First).
6. Routing/composition? → `app` layer, thin.

## Tooling

- Linter: **Steiger** (`npx steiger src`) — checks import rule, public API usage, cross-imports, structure. Run it when validating a project.
- Naming: stick to standard layer/segment names exactly.

## Anti-patterns to flag

- Extracting to `features`/`entities` when the code is used in only one place (no confirmed reuse yet).
- A feature importing another feature, or an entity importing an entity without `@x`.
- Business logic in `shared`.
- `components/`, `hooks/`, `utils/` as segment names.
- Importing from a slice's internals instead of its `index.ts`.
- Recreating the deprecated `processes` layer.

## Non-standard or partially-migrated projects

If the project does not fully follow FSD (e.g. mixed `components/` + partial layers, or legacy flat structure), do not refuse to help. Instead:
1. Identify which parts already align with FSD layers and work within them.
2. For new code, place it correctly per FSD rules regardless of surrounding legacy code.
3. Flag the deviation clearly: "This file is outside FSD structure — ideally it belongs in `features/auth`. I'll place new code correctly and you can migrate the old file separately."
4. Never silently place new code in a non-FSD location just because the project is inconsistent.

**Migration priority order** (highest impact first):
- `shared/` — foundational, everything depends on it; migrate or establish this first
- `entities/` — widely imported; correct boundaries here reduce downstream violations
- `features/` — frequently modified; high payoff for import rule compliance
- `pages/` and `widgets/` — migrate last; they compose the layers above

When legacy and FSD structures conflict on the same file, prefer the FSD placement for any new code and note the legacy file as a migration candidate.