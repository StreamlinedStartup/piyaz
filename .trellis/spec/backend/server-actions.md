# Server Actions

> Pattern reference: `lib/actions/profile.ts`, `lib/actions/team-invite-code.ts`, `lib/actions/rate-limit-action.ts`.

Server actions are the auth-checked mutation entry points for the UI. They live in `lib/actions/*.ts`, start with `"use server"`, and never touch the DB directly (that is `lib/data/`'s job).

## Canonical Shape

```ts
"use server";

import { z } from "zod/v4";
import { requireSession } from "@/lib/auth/session";
import { requireLegalConsent } from "@/lib/auth/consent";
import { checkActionRateLimit } from "@/lib/actions/rate-limit-action";

const updateProfileSchema = z.object({
  name: z.string().transform((value, ctx) => { /* normalize or ctx.addIssue */ }),
});

export async function updateProfileAction(input: { name: string }) { ... }
```

## Rules

1. **`"use server"` at the top**, imports from `@/lib/auth/*`, `@/lib/data/*`, and sibling action helpers only.
2. **Validate input with zod v4** (`import { z } from "zod/v4"`), using `.transform` + `ctx.addIssue` for policy normalization (see `updateProfileSchema` in `lib/actions/profile.ts:31`, which routes display names through `normalizeDisplayName`). Shared parse helpers like `parseOrFail` live in `lib/actions/team-errors.ts`.
3. **Return typed result objects, never throw to the client.** The convention is a discriminated union with a closed failure-code set:

   ```ts
   // lib/actions/team-invite-code.ts
   export type InviteCodeResult =
     | { ok: true; data: InviteCodeMetadata }
     | { ok: false; code: "unauthorized" | "forbidden" | "invalid_input"
                        | "not_found" | "rate_limited" | "unknown"; message: string };
   ```

   Components branch on `result.ok` and render `result.message` inline.
4. **Rate limit every mutation** via `lib/actions/rate-limit-action.ts`. Two options, both two-key (per-user AND per-IP):
   - `checkActionRateLimit(config, userId)` when the action manages its own flow, mapping failure to the `rate_limited` code.
   - `authorizeWrite(config)` for the flood-safe ordering (IP limb before the session lookup, user limb after); it returns the `AuthContext` and throws `RateLimitError` / consent errors for the caller's boundary to map. The graph mutation wrappers (`lib/graph/mutations.ts`) use this.
   Default `backendKind` is `"actions"`; only route to `"auth"` if the limit is exactly 5/60 (see the comment on `backendKind` in `rate-limit-action.ts:28`).
5. **Auth + consent before work.** `requireSession()` / `getAuthContext()` plus `requireLegalConsent(userId)` (or `assertLegalConsent` inside `authorizeWrite`). Admin-scoped actions resolve an admin context helper first (see `resolveAdminContext` in `lib/actions/team-invite-code.ts`).
6. **Map external errors, do not leak them.** better-auth failures go through `mapBetterAuthError` (`lib/actions/team-errors.ts`); unique-key races through `isUniqueViolation` (`lib/db/errors.ts`).
7. **Hide internal fields** by mapping rows to public-facing types before returning (see `InviteCodeMetadata` + `toMetadata` in `team-invite-code.ts`).
8. **Cache invalidation** uses `revalidatePath` from `next/cache` where the mutation changes RSC-rendered data (`lib/actions/profile.ts`).
9. **Deferred work** after the response uses `deferRequestWork` from `@/lib/db/request-store`, not floating promises.

## Failure-code discipline

Keep failure codes machine-actionable and per-domain (`InviteCodeFailureCode`, `JoinFailureCode`, `TeamActionResult`). When adding a new action, reuse the domain's existing result type and messages map (`TEAM_ACTION_MESSAGES`) before inventing a new one.
