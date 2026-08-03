# Type Safety

> Type safety patterns in Piyaz. Pattern references: `lib/auth/context.ts`, `lib/actions/team-invite-code.ts`, `lib/query/queries.ts`.

---

## Type Organization

- Server view types are the single source of truth and flow to the client: components and query fetchers import `ProjectGraphSlim`, `TaskFullWithEdges`, `NoteFullResult`, etc. from `@/lib/data/...`. Do not redeclare response shapes client-side.
- Domain unions (`ProjectStatus`, `TaskStatus`) live with the data layer; UI code imports them.
- Cross-cutting client types live in `lib/types.ts`; component-local prop interfaces stay in the component file.

---

## Validation

- Runtime validation is zod v4, imported as `import { z } from "zod/v4"` (never bare `"zod"`), and it happens at trust boundaries: server actions and MCP tool schemas. Client components do light UX-level checks (trim, empty) but rely on the action's schema for real validation.
- Policy normalization lives inside schemas via `.transform` + `ctx.addIssue` (see `updateProfileSchema` in `lib/actions/profile.ts:31`).

---

## Common Patterns

- **Discriminated result unions** for anything that can fail: `{ ok: true; data } | { ok: false; code; message }` with a closed, per-domain failure-code union (`InviteCodeResult`). Components narrow on `result.ok`.
- **Branded types** make unverified values unrepresentable: `AuthContext` (`lib/auth/context.ts`) can only be minted by the auth layer; `Tx` / `Conn` (`lib/db/raw.ts`) reject un-scoped DB handles. If you add a value that must only come from a gatekeeper, brand it the same way (unique-symbol property).
- **Named actor/source unions** over booleans: `ActorDescriptor = { source: "web" | "mcp" | "system"; ... }`.
- `as const` key factories give literal tuple types for query keys (`lib/query/keys.ts`).

---

## Forbidden Patterns

- `any` and unchecked casts in app code. The one sanctioned cast shape is the test-only `globalThis as unknown as { __setTestSession ... }` bridge.
- Re-typing server payloads by hand on the client; import the view type.
- Local casts on raw payload fields in two places; lift a shared type instead (this is the cross-layer guide's trigger in `.trellis/spec/guides/`).
