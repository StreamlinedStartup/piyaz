# Quality Guidelines

> Code quality standards, enforced by ESLint (`eslint.config.mjs`), Biome (format), and tsc.

---

## Forbidden Patterns

These are lint **errors** with the reason in the rule message (`eslint.config.mjs:80-159`); the short list:

- Importing `@/lib/db` (or `@/lib/db/connection`) outside the data ring; application code imports `@/lib/data`.
- Bare `db.select/insert/update/delete`, `db.query.*`, `db.transaction()`, `.batch()`, `.execute()`; all DB access goes through `withUserContext` / `withUserContextRead` / named `lib/db/raw/` functions.
- `serviceRoleDb.*` outside the documented allowlist (BYPASSRLS).
- Importing `@cloudflare/workers-types` (clobbers DOM types); declare minimal local stubs like `lib/realtime/broker-do.ts`.
- `new Request(<Request>, init)`; pass a URL string (Cloudflare shim throws otherwise); see `app/api/auth/oauth2/token/route.ts`.
- No real project data, ids, emails, or tokens anywhere; this repository is public.

Frontend-specific:

- No new state-management or form libraries; the stack is `useState` + TanStack Query + server actions.
- No hard-coded colors; use the token classes from `app/globals.css`.
- No default exports for components.

---

## Required Patterns

- JSDoc on every exported function/component (existing code is fully documented; match it).
- Typed result unions from server actions, rendered as inline errors.
- Query keys from `lib/query/keys.ts` factories.
- Runtime pairs: if you touch a `.node.ts` module, check its `.workers.ts` sibling.

---

## Testing Requirements

New behavior gets a `bun test` case in the matching `tests/<layer>/` directory (see `backend/testing.md` for fixtures, session mocking, and DB setup). UI-adjacent logic is tested at the layer below (actions/data/api); there is no browser test harness in the repo today, so do not add one ad hoc.

---

## Code Review Checklist

1. `bun run lint && bun run typecheck && bun run format:check && bun run test` all pass.
2. Any DB touch goes through the data layer with an `AuthContext`.
3. New endpoint or fetcher honors the ETag/304 contract.
4. Errors are mapped to typed codes, not thrown to the UI; 500 bodies stay opaque outside dev.
5. Conventional commit, imperative, lowercase, under 72 chars; versions/CHANGELOG are release-please's job.
