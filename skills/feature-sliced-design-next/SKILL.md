---
name: feature-sliced-design-next
description: Apply Feature-Sliced Design (FSD) v2.1 architecture to Next.js App Router projects. Use this skill ALWAYS when working in a Next.js codebase and the user mentions FSD, "feature-sliced", layers, slices, segments, code structure, or asks where a piece of code should live; when scaffolding a new Next.js app; when refactoring; when resolving cross-imports or import-rule violations; or when deciding between app/, pages/, widgets/, features/, entities/, and shared/. Trigger even if the user does not say "FSD" explicitly but is organizing a Next.js project into layers or asking architecture/placement questions.
---

# Feature-Sliced Design v2.1 — Next.js App Router

FSD is a layered architecture for frontend apps. This skill is the Next.js App Router variant. Core FSD rules are universal; the Next.js-specific part is solving the `app/` routing-vs-FSD conflict (see "Next.js integration" below).

## The one rule that matters most: the import rule

A module in a slice can import only from layers **strictly below** it. Same-layer and upward imports are forbidden (the only exception is the `@x` notation for entities, below).

```
app        → can import: pages, widgets, features, entities, shared
pages      → can import: widgets, features, entities, shared
widgets    → can import: features, entities, shared
features   → can import: entities, shared
entities   → can import: shared   (+ other entities only via @x)
shared     → can import: nothing above it
```

If you ever want a feature to import another feature, or an entity to import another entity directly — stop. That is the signal to either lift the composition up a layer (compose both in a widget/page) or use `@x` (entities only).

## Layers (top to bottom)

1. **app** — app initialization: providers, global styles, root layout wiring. In Next.js this is `src/app`, NOT the routing `app/`.
2. **pages** — full pages or large self-contained route segments. Each page is a slice. Pages cannot import other pages.
3. **widgets** — large self-contained UI/logic blocks delivering a complete use case (e.g. a header, a comments section). Use widgets when a reusable block needs layers above shared.
4. **features** — *reused* user interactions that bring business value (e.g. `auth/login`, `add-to-cart`). Not everything is a feature — only extract when reused.
5. **entities** — business entities the app works with (`user`, `product`, `order`). Reusable domain reads + domain UI primitives.
6. **shared** — reusable, business-agnostic code: UI kit, API client, config, lib, router primitives. No business logic, no upper-layer imports.

`processes` is **deprecated** — do not create it.

`app` and `shared` have **no slices** — they go straight to segments. `pages`, `widgets`, `features`, `entities` MUST have slices, and segments live inside slices.

## Slices and segments

- **Slices** partition a layer by business domain (`entities/user`, `features/auth`). Slices on the same layer cannot import each other.
- **Segments** partition a slice by technical purpose. Conventional names:
  - `ui` — components, styles, formatters
  - `api` — backend interactions, request functions, mappers
  - `model` — schemas, stores, business logic, types
  - `lib` — internal helper code for this slice
  - `config` — feature flags, constants
- Segment names describe **purpose (why), not essence (what)**. Never `components/`, `hooks/`, `utils/` as segments — they force people to read every file.

## Public API (mandatory)

Every slice exposes a contract via `index.ts`. Outside code imports only from the slice root, never from internal files.

```ts
// features/auth/index.ts
export { LoginForm } from "./ui/LoginForm";
export { useAuth } from "./model/useAuth";
```

For **shared**, define a public API per segment (`shared/ui`, `shared/api`) rather than one giant index — keeps imports organized by intent.

## Pages First (the v2.1 mental model)

Default to placing code **where it is used**. Start in `pages` and `widgets`. Extract to `features`/`entities` ONLY when real reuse appears — do not predict the future.

- A form used on one page → lives in that page's `ui` segment.
- The same interaction appears on a second page → now extract to `features`.
- Premature `entities/`/`features/` is the most common FSD mistake. An entity slice that is imported in exactly one place is a smell.

Caveat for multi-project / shared-backend work: if you KNOW a slice (e.g. `auth`) is shared across multiple apps in a monorepo, extracting early is justified — "reuse" there is cross-package, not in-app. Judge reuse at the right scope.

## `@x` notation — the ONLY way entities cross-import

Entities legitimately reference each other (an `Order` has a `User`). Don't work around it; make it explicit.

```
entities/
  user/
    @x/
      order.ts        ← public API exposed specifically TO the order entity
    index.ts          ← regular public API
    model/user.ts
  order/
    model/order.ts
```

```ts
// entities/user/@x/order.ts
export type { User } from "../model/user";

// entities/order/model/order.ts
import type { User } from "entities/user/@x/order";
```

Read `A/@x/B` as "A crossed with B". Rules: entities layer only, keep to a minimum, treat as last resort. If you're reaching for `@x` a lot, your entities are probably split too granularly — consider merging them.

## Next.js App Router integration (the critical conflict)

Next.js wants `app/` to mirror URLs. FSD wants a flat slice structure. They collide. The standardized solution:

```
project-root/
├── app/                  ← Next.js routing ONLY (thin route files)
│   ├── (public)/
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── (auth)/
│   │   ├── login/page.tsx
│   │   └── register/page.tsx
│   └── api/.../route.ts
├── pages/                ← MUST exist (can be empty + README) or Next tries src/pages as Pages Router and breaks the build
└── src/
    ├── app/              ← FSD app layer (providers, global styles)
    ├── pages/            ← FSD pages layer (real page logic)
    ├── widgets/
    ├── features/
    ├── entities/
    └── shared/
```

Key points:
- The Next.js `app/` folder stays at project root and is for **routing only**. Route files (`page.tsx`) must be thin — they re-export / delegate to `src/pages/**`.
- Add an empty root `pages/` folder with a `README.md` explaining why it exists (prevents Next from treating `src/pages` as the Pages Router).
- `middleware.ts` and `instrumentation.ts`, if used, go in **project root** alongside `app/` and `pages/`.
- Route files import from `src/pages` and below — never from deep internals.

Thin route file example:
```tsx
// app/(public)/page.tsx
export { HomePage as default } from "@/pages/home";
```

Server Components / Server Actions placement:
- Routes = composition and wiring only.
- Features own mutations, Server Actions, revalidation, client interactivity.
- Entities own domain reads and domain UI primitives.
- Pages finalize a route segment's UI by composing widgets/features/entities.

## API requests

- Shared, reusable requests → `shared/api` (a `client.ts` wrapper + `endpoints/` grouped by endpoint, re-exported from `index.ts`).
- Request used by exactly one slice → that slice's `api` segment; no need to expose it in the slice's public API.
- Do NOT prematurely dump API response types into `entities` — backend shapes differ from frontend needs. Transform in `shared/api` or the slice `api` segment, keep entities frontend-focused.
- OpenAPI backend → generate types/clients (orval, openapi-typescript) into `shared/api/openapi` with a README documenting regeneration.

## Auth & token storage

- Login/register pages → one `pages/login` slice (group both forms, they're similar). A login dialog usable anywhere → a widget instead.
- Client-side validation schema → `pages/login/model/` using Zod, exposed to the `ui` segment.
- Token storage preference: **cookies** (esp. with Next.js server side — keep cookie infra in `shared/api`). If manual storage is required, store either in `shared` (plays well with a shared API client + refresh middleware) or in an `entities/user` (or current-user/"viewer") model store. Do NOT store app-wide tokens in pages/widgets.

## State libraries (TanStack Query / similar)

Share cache keys, query/mutation option factories, and API data types via `shared`. Co-locate query hooks with the slice that owns the data (often `entities/<x>/api` or the consuming feature). Keep keys centralized so invalidation stays consistent.

## Decision checklist (where does this code go?)

1. Is it business-agnostic infra (button, fetch wrapper, date util)? → `shared`.
2. Is it a business entity's data/UI primitive, reused? → `entities/<name>`.
3. Is it a reused user interaction with business value? → `features/<name>`.
4. Is it a big reusable UI block needing layers above shared? → `widgets/<name>`.
5. Is it specific to one route and not (yet) reused? → `pages/<name>` (Pages First).
6. Does a route file contain logic? → move logic into `src/pages/**`, keep the route thin.

## Tooling

- Linter: **Steiger** (`npx steiger src`) — checks import rule, public API usage, cross-imports, structure. Recommend running it when validating a project.
- Naming: stick to the standard layer/segment names exactly; prefix with "FSD" when discussing with the team to avoid collisions (FSD#page vs a log "page").

## Anti-patterns to flag

- Premature extraction into `features`/`entities` (used in one place).
- A feature importing another feature, or an entity importing an entity without `@x`.
- Business logic in `shared` or in Next.js route files.
- `components/`, `hooks/`, `utils/` used as segment names.
- Importing from a slice's internals instead of its `index.ts`.
- Recreating the deprecated `processes` layer.