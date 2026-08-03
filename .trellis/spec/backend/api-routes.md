# API Route Handlers

> Pattern reference: `app/api/task/[taskId]/route.ts`, `app/api/projects/route.ts`.

Route handlers live under `app/api/**/route.ts` and export `GET` / `HEAD` (and other verbs) that delegate to a single private `handle()` function so both verbs share auth and 304 logic.

## Canonical Shape

```ts
// app/api/projects/route.ts (abridged)
import { getAuthContext } from "@/lib/auth/context";
import { conditionalRespond, etagMatches } from "@/lib/api/conditional";
import { internalError } from "@/lib/api/error";
import { error } from "@/lib/api/response";
import { consentGateResponse } from "@/lib/auth/consent";

async function handle(req: Request): Promise<Response> {
  let ctx;
  try {
    ctx = await getAuthContext();
  } catch {
    return error("Unauthorized", 401);
  }

  const gate = await consentGateResponse(ctx.userId);
  if (gate) return gate;

  try {
    const max = await getProjectListMaxUpdatedAt(ctx);
    if (req.method === "HEAD" || etagMatches(req, max)) {
      return conditionalRespond(req, null, max);
    }
    const page = await listProjectsSlim(ctx, { cursor, limit });
    return conditionalRespond(req, page, max);
  } catch (err) {
    return internalError("projects", err);
  }
}

export async function GET(req: Request) { return handle(req); }
export async function HEAD(req: Request) { return handle(req); }
```

## Rules

1. **Auth first.** Resolve `getAuthContext()` (from `lib/auth/context.ts`) and map failure to `error("Unauthorized", 401)`. The context is branded; pass it down instead of raw user ids.
2. **Consent gate second.** `consentGateResponse(ctx.userId)` returns a 403 response when legal re-acceptance is outstanding; return it verbatim.
3. **Cheap validator probe before heavy reads.** Fetch the max `updatedAt` (or run `assertTaskAccess`) first so `HEAD` and `If-None-Match` matches short-circuit with `conditionalRespond(req, null, validator)` before paying the join cost. See the ordering comment in `app/api/task/[taskId]/route.ts:45`.
4. **Every public data-layer call authorizes itself.** Routes still call ctx-taking data functions (e.g. `getTaskFullWithEdges(ctx, taskId)`) even after an access assertion; do not bypass that with "already checked" shortcuts.
5. **Error mapping.** `ForbiddenError` from `lib/auth/authorization.ts` maps to a 404 ("Task not found"), never a 403 that would confirm existence. Everything else falls to `internalError("<route-label>", err)`.
6. **Responses** come from `lib/api/response.ts` (`ok`, `error`) and `lib/api/conditional.ts`; do not hand-build `NextResponse.json` in routes.
7. **Query params** are parsed defensively inline (see the `limit` clamp in `app/api/projects/route.ts:44`); the data layer clamps again.
8. **Document the handler** with a JSDoc block explaining the caching/authorization ordering; every existing route does.

## Realtime side effects

Routes that feed the workspace may register SSE subscriptions, but only for callers with a live connection: `if (broker.hasConnections(ctx.userId)) broker.register(...)` with an explicit TTL constant (`app/api/task/[taskId]/route.ts:60`). Never register unconditionally; connection-less registrations leak broker memory.
