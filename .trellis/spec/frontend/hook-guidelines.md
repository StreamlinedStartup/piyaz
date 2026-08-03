# Hook Guidelines

> How hooks are used in Piyaz. Pattern references: `hooks/`, `lib/query/queries.ts`.

---

## Custom Hook Patterns

- Reusable hooks live in `hooks/`, one hook per file, named `use<Thing>`: `useInlineEdit`, `useCopyToClipboard`, `useMediaQuery`, `useModalChrome`, `usePopoverAnchor`, `useUndo`.
- Hooks are small and single-purpose (most are under 150 lines). Behavior that spans components (inline-edit activation, popover anchoring, undo stacks) belongs here; single-component state stays in the component.
- A hook returns either a value, or a props-bag object designed to spread onto elements (`useInlineEdit` returns `triggerProps`, `onEditorFocus`, `onActivate`).
- `.tsx` extension only when the hook returns JSX (`useUndo.tsx`).

---

## Data Fetching

TanStack Query v5 is the data-fetching layer. The pieces are already built; do not fetch ad-hoc in components:

- **Keys** come from the factories in `lib/query/keys.ts` (`projectKeys.graph(projectId)`, `taskKeys`, `noteKeys`, `teamKeys`, `myTasksKeys`). Never inline a key array; compound prefixes exist so `invalidateQueries({ queryKey: ["task", projectId] })` can drop a project's task entries wholesale.
- **Fetchers** come from `lib/query/queries.ts` (`fetchProjectsPage`, `fetchProjectGraph`, `fetchTaskBody`, ...). They wrap `conditionalFetch` / `conditionalFetchPage` (`lib/query/conditional-fetch.ts`), which implement the ETag/304 contract the API routes expose.
- **Response types** are imported from the data layer's view types (`ProjectGraphSlim`, `TaskFullWithEdges` from `@/lib/data/views`), not re-declared client-side.
- Mutations go through server actions (see component-guidelines), then invalidate or update caches via helpers such as `removeProjectFromList` in `lib/query/queries.ts` and `lib/query/note-cache.ts`.

---

## Naming Conventions

- `use` prefix, camelCase, file named after the hook.
- Cache-manipulation helpers are verbs, not hooks (`removeProjectFromList(queryClient, ...)`), and take the `QueryClient` explicitly.

---

## Common Mistakes

- Inlining query keys instead of using the factories; invalidation then silently misses entries.
- Bypassing `conditionalFetch` with raw `fetch`: you lose the 304 handling the API is designed around and re-download unchanged payloads.
