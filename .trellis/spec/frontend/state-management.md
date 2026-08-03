# State Management

> How state is managed in Piyaz. Pattern references: `lib/query/`, `lib/realtime/`, `components/providers/`.

---

## State Categories

| Category | Tool | Example |
|----------|------|---------|
| Local UI state | `useState` / render-phase derivation | edit/error/loading flags in `TitleSection.tsx` |
| Shared UI state | React context providers in `components/providers/` | theme, workspace chrome |
| Server state | TanStack Query v5 (`lib/query/`) | project list, graph, task bodies, notes |
| Realtime updates | SSE via `/api/events` + broker subscriptions | task/project change fan-out |

There is no Redux/Zustand/Jotai store. Do not introduce one; promote to a provider in `components/providers/` only when prop drilling crosses several layers.

---

## Server State

- The `QueryClient` is configured in `lib/query/client.ts`; components consume it through the provider.
- Query keys, fetchers, and types: see [Hook Guidelines](./hook-guidelines.md). The server pairs every list/detail endpoint with an ETag validator, and `conditionalFetch` (`lib/query/conditional-fetch.ts`) turns 304s into cache reuse. Any new endpoint you consume should follow the same contract.
- Pagination is keyset-based: pages are `{ rows, nextCursor }` (`ProjectListPage` in `lib/query/queries.ts`), consumed as `InfiniteData`.
- The workspace fetches heavy fields lazily: the slim project graph omits description/plan/criteria, and selecting a task fetches `/api/task/[id]` (documented in `app/api/task/[taskId]/route.ts:16`). Keep new features on this slim-graph + lazy-detail split.

---

## Realtime

Server-sent events flow through the broker (`lib/realtime/broker.ts`): API reads register short-TTL subscriptions for connected users, and the client reacts to events by invalidating the matching query keys. When adding a realtime-aware feature, register subscriptions server-side only for users with live connections (`broker.hasConnections`) and reuse the existing event types in `lib/realtime/events.ts`.

---

## When to Use What

1. Start with component-local `useState`.
2. Sync props to state at render time, not in `useEffect` (see the `syncedInitialTitle` pattern in `TitleSection.tsx:33`).
3. Anything fetched from the server belongs in TanStack Query with a factory key; never mirror server data into local state stores.
4. After a mutation, update the cache surgically (helpers in `lib/query/note-cache.ts`) or invalidate by key prefix; do not refetch the world.

---

## Common Mistakes

- Holding server data in `useState` "for editing" and letting it drift from the cache; keep the query cache canonical and derive the editor's initial value from it.
- Forgetting that any change anywhere revalidates the whole project list (the ETag is the global max `updated_at`, `app/api/projects/route.ts:16`); per-row cache surgery is an optimization, not a correctness requirement.
