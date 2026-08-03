# Directory Structure

> How frontend code is organized in Piyaz (Next.js App Router).

---

## Directory Layout

```
app/                    # Routes, RSCs, route handlers
├── (auth)/             # Route group: sign-in/up, reset, verify
├── (public)/           # Route group: public marketing/legal pages
├── api/                # Route handlers (see backend/api-routes.md)
├── settings/
│   └── _components/    # Route-private client components (underscore = not a route)
├── project/            # Workspace routes
├── layout.tsx          # Root layout, fonts, providers
└── globals.css         # Tailwind v4 entry + design tokens

components/             # Shared client components, grouped by product area
├── auth/               # Auth forms + AuthShell primitives
├── graph/              # Dependency-graph canvas
├── home/               # Home grid (ProjectCard, TeamFilterBar)
├── layout/             # App chrome
├── my-tasks/
├── providers/          # Client-side context providers
├── shared/             # Cross-area primitives (Badge, Avatar, EditButton, AutoGrowTextarea)
└── workspace/          # Workspace panels (project-settings/, task views)

hooks/                  # Reusable hooks (useInlineEdit, useMediaQuery, useUndo, ...)
lib/query/              # TanStack Query: client, key factories, fetchers, conditional fetch
lib/auth-client.ts      # better-auth client (signIn, signUp) for auth forms
assets/, public/        # Static assets
```

---

## Module Organization

- Components used by exactly one route live in that route's `_components/` folder (e.g. `app/settings/_components/team-manage/InviteSection.tsx`). Promote to `components/<area>/` only when a second route needs them.
- Area folders under `components/` map to product surfaces, not technical kinds. A workspace settings panel goes in `components/workspace/project-settings/`, not a generic `forms/` folder.
- Data-fetching helpers do not live next to components; they live in `lib/query/queries.ts` and are imported by hooks/components.

---

## Naming Conventions

- Component files: `PascalCase.tsx`, named exports (no default exports): `TitleSection.tsx` exports `TitleSection`.
- Hooks: `useCamelCase.ts` in `hooks/` (`useInlineEdit.ts`); `.tsx` only when the hook renders JSX (`useUndo.tsx`).
- Non-component modules inside component folders are `kebab-case.ts` (`components/auth/turnstile-state.ts`).
- Route-private folders start with `_` so Next.js skips them as segments.

---

## Examples

- Well-organized area: `components/workspace/project-settings/` (one focused section component per concern: `TitleSection`, `StatusSection`, `CategoriesSection`).
- Auth surface: `components/auth/` composes small primitives (`AuthInput`, `AuthSubmit`, `AuthShell`) into forms (`SignInForm`, `SignUpForm`).
