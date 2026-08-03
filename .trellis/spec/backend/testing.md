# Testing

> Pattern reference: `tests/api/task.test.ts`, `tests/setup/`, `bunfig.toml`.

Tests run with `bun test` against a real Postgres started from `docker-compose.test.yml`. There is no mocked DB; RLS policies are part of what the suite verifies.

## Running

```sh
bun run test                      # pretest starts the DB unless TEST_DATABASE_URL is set
bun test tests/api/task.test.ts   # one file, once the test DB is up
bun run db:test:down              # tear down the DB and volume
```

`bunfig.toml` preloads `tests/setup/preload.ts` and caps each test at 15 s; anything slower is treated as a hang, not a slow test.

## Directory layout

Tests mirror the source layers: `tests/api/`, `tests/actions/`, `tests/data/`, `tests/db/`, `tests/auth/`, `tests/graph/`, `tests/mcp/`, `tests/security/`, `tests/realtime/`, `tests/ui/`, plus `tests/regression/` for pinned bug reproductions. Put a new test next to its layer, named `<module>.test.ts`.

## Canonical test shape

```ts
// tests/api/task.test.ts (abridged)
import { test, expect, afterEach } from "bun:test";
import { truncateAll } from "@/tests/setup/schema";
import { seedUserOrgProject } from "@/tests/setup/seed";
import { superuserPool } from "@/tests/setup/global";
import { GET } from "@/app/api/task/[taskId]/route";

const setSession = (globalThis as unknown as {
  __setTestSession: (s: { user: { id: string } } | null) => void;
}).__setTestSession;

afterEach(async () => {
  await truncateAll();
});

test("GET /api/task/[id] — ...", async () => {
  const f = await seedUserOrgProject("task-noconn");
  setSession({ user: { id: f.userId } });
  const res = await GET(new Request(`http://test/api/task/${id}`), {
    params: Promise.resolve({ taskId: id }),
  });
  expect(res.status).toBe(200);
});
```

## Conventions

1. **Route handlers are invoked directly** (import `GET`/`POST` from the route module, build a `Request`, pass `params` as a resolved Promise). No HTTP server is started.
2. **Sessions are faked via the preload hook**: `globalThis.__setTestSession` (wired in `tests/setup/preload.ts`) swaps the better-auth session; call it per test, reset implicitly by preload.
3. **Fixtures come from `tests/setup/seed.ts`** (`seedUserOrgProject`, `seedSecondMember`), each taking a suffix string to keep rows distinguishable. Ad-hoc rows are inserted with the tagged-template `superuserPool()` client; call `sql.end({ timeout: 5 })` in a `finally` for one-off clients.
4. **Clean up in `afterEach`** with `truncateAll()` from `tests/setup/schema.ts`, plus any module-level reset hooks (e.g. `broker._resetForTests()`).
5. **Pick the pool matching the prod path's privilege**: `appUserConnect()` (NOBYPASSRLS) to verify policies actually fire, `serviceRoleConnect()` only where prod also bypasses RLS (`tests/setup/seed.ts` documents this).
6. **Regression tests carry a comment naming the bug they pin** (see the broker-leak comment in `tests/api/task.test.ts`).
7. Test names read as behavior sentences: `"GET /api/task/[id] — registers a 10-min TTL sub when the user has a live connection"`.
