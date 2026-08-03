# Backend Guidelines

> Server-side conventions for Piyaz, derived from the current codebase. Every rule below is backed by an existing code example; follow the cited file when in doubt.

---

## Layer Map

| Layer | Location | Role |
|-------|----------|------|
| Route handlers | `app/api/**/route.ts` | HTTP surface: auth, consent gate, conditional GET, delegate to data layer |
| Server actions | `lib/actions/*.ts` | Auth-checked mutation entry points for the UI |
| Data layer | `lib/data/*.ts` | The ONLY layer that touches `withUserContext` / RLS-scoped DB access; one module per aggregate |
| DB ring | `lib/db/` | Drizzle client, driver selection, RLS helpers, raw statement registry (`lib/db/raw/`) |
| Auth | `lib/auth/` | better-auth wiring, sessions, branded `AuthContext`, consent, permissions |
| MCP / graph | `lib/mcp/`, `lib/graph/tools/` | Tool registration and schemas; one handler module per tool |
| Realtime | `lib/realtime/` | SSE broker; `.node.ts` / `.workers.ts` runtime pair |

## Guides

| Guide | Covers |
|-------|--------|
| [API Routes](./api-routes.md) | Route handler shape: auth, consent, ETag/304, error mapping |
| [Server Actions](./server-actions.md) | `"use server"` actions: zod input, typed results, rate limits |
| [Data Layer & RLS](./data-layer-rls.md) | `withUserContext`, forbidden DB calls, `serviceRoleDb` allowlist |
| [Testing](./testing.md) | `bun test` against a real Postgres, seed fixtures, session mocking |

---

## Logging

There is no logging framework (no winston/pino). The convention is `console.error` with a scoped label, centralized behind helpers:

- Route handler 500s go through `internalError(label, err)` in `lib/api/error.ts:41`, which logs `[label] error:` server-side and returns an opaque `{ error: "Internal error" }` body outside `NODE_ENV === "development"`.
- Auth router errors go through `logAuthApiError` in `lib/auth/api-error-log.ts`, which serializes the full `cause` chain because better-auth hides stacks.
- Never log secrets, tokens, or emails; this repository is public and `lib/auth/api-error-log.ts` scrubs its output.

Do not introduce a logging library without an explicit task for it.

## Two Build Targets

Self-host Node and Cloudflare Workers build from one codebase. Runtime-specific modules come in `.node.ts` / `.workers.ts` pairs (e.g. `lib/realtime/_broker.node.ts` / `_broker.workers.ts`); the bare module re-exports the Node variant and `next.config.ts` swaps it for Workers builds. Editing one variant usually means editing its sibling, because typecheck only sees `.node`.

## CI Gates (run locally before finishing)

```sh
bun run lint        # eslint, includes the RLS restriction rules
bun run typecheck   # tsc --noEmit
bun run format:check
bun run test        # starts the test DB if TEST_DATABASE_URL is unset
```

CI order: `bun audit`, `format:check`, `lint`, `typecheck`, `db:generate` diff check, `test`, `check:plugins`.
