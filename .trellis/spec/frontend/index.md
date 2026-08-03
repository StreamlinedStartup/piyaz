# Frontend Development Guidelines

> Conventions for Piyaz's Next.js App Router frontend, derived from the current codebase. Backend/server conventions live in [`../backend/`](../backend/index.md).

---

## Guidelines Index

| Guide | Description | Status |
|-------|-------------|--------|
| [Directory Structure](./directory-structure.md) | `app/` vs `components/<area>` vs `hooks/`, naming | Filled |
| [Component Guidelines](./component-guidelines.md) | Client components, props JSDoc, forms, Tailwind tokens | Filled |
| [Hook Guidelines](./hook-guidelines.md) | `hooks/` conventions, TanStack Query fetch layer | Filled |
| [State Management](./state-management.md) | Local state, query cache, SSE realtime | Filled |
| [Quality Guidelines](./quality-guidelines.md) | Lint-enforced forbidden patterns, review checklist | Filled |
| [Type Safety](./type-safety.md) | Server view types, zod v4 boundaries, branded types | Filled |

---

## Stack Snapshot

- Next.js App Router (webpack build), React server components by default, `"use client"` where interactive.
- Tailwind CSS v4 with semantic tokens in `app/globals.css`; fonts via `@fontsource-variable` (Inter, Geist Mono).
- TanStack Query v5 for server state; better-auth client for auth flows; no other state or form libraries.
- Two deploy targets (Node self-host, Cloudflare Workers); frontend code is target-agnostic, runtime forks live in `lib/**` `.node.ts` / `.workers.ts` pairs.

Each guide cites the file it was derived from. If code and guide disagree, the code is ground truth; fix the guide in the same task.

---

**Language**: All documentation should be written in **English**.
