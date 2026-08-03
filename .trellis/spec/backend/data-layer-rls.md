# Data Layer & Row-Level Security

> Pattern reference: `lib/data/project.ts`, `lib/db/rls.ts`, `eslint.config.mjs:80-159`.

Row-level security is the tenant boundary. The `lib/data/` ring is the only layer allowed to touch the DB, and it does so exclusively through the RLS wrappers.

## Rules (ESLint-enforced; the linter states each reason inline)

1. **Never call `db.select` / `db.insert` / `db.update` / `db.delete`, `db.query.*`, `db.transaction()`, or bare `.batch()`.** Under `app_user` with no GUC these default-deny and silently return empty or wrong-tenant data. Use:
   - `withUserContext(userId, async (tx) => ...)` for writes and transactional reads,
   - `withUserContextRead(userId, (read) => [...statements])` for the neon-http read batch path,
   both from `@/lib/db/rls`.
2. **`serviceRoleDb` is BYPASSRLS and allowlisted per call site.** Current allowed sites are enumerated in the lint messages (`lib/data/oauth-session.ts`, parts of `lib/data/account.ts`, `lib/data/membership.ts` admin lookups). A new bypass site must first be audited against a `SECURITY DEFINER` function in `docker/rls-functions.sql`.
3. **No direct `.execute()`.** Raw SQL lives as a named function in `lib/db/raw/` (e.g. `lib/db/raw/get-project-list-max-updated-at.ts`) and is invoked via `executeRaw` / `executeRawDiscard`.
4. **Application code imports `@/lib/data`, never `@/lib/db`.** Only the data ring and security gates are on the `DB_IMPORT_ALLOWLIST` in `eslint.config.mjs:163`.
5. **Handles are branded.** `Tx` (an `RlsTx`) and `Conn` from `lib/db/raw.ts` reject bare `db.transaction()` handles at the type level; see `tests/security/conn-brand.test.ts`.

## Module conventions

- `import "server-only"` at the top of every data module.
- One module per aggregate (`lib/data/task.ts`, `lib/data/project.ts`, `lib/data/note.ts`); shared column fragments get their own module (`lib/data/edge-columns.ts`).
- Public functions take `(ctx: AuthContext, ...)` and authorize themselves via a fresh membership JOIN or an `assert*Access` helper; callers never pre-authorize on their behalf.
- `withUserContext` validates the `userId` is a UUID and throws `InvalidUserIdError`, which the action layer maps to `invalid_input` (`lib/db/rls.ts`).
- Keyset pagination uses `encodeCursor` / `decodeCursor` from `lib/data/cursor.ts`; limits are clamped in the data layer.
- Activity trails are written in the same transaction via `insertActivityEvents` (`lib/data/activity.ts`).

## Schema ownership

The `public` schema belongs to Drizzle (`lib/db/schema.ts` + `bun run db:generate`). The `piyaz_auth` schema, roles, grants, and RLS policies are hand-written SQL under `docker/`. Migrations are roll-forward only; `db:push` is for throwaway test DBs only (guarded by `scripts/assert-throwaway-db.ts`).
