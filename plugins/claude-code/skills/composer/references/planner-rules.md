# Planner rules (composer Phase 2 extract)

Condensed extract of the canonical piyaz references for the composer
planner. Sources: `skills/piyaz/references/conventions.md` §1,
`skills/piyaz/references/artifacts.md` §1 (`description`,
`acceptanceCriteria`, `decisions`), §6, and
`skills/piyaz/references/lifecycle.md` §1 (Summary, `draft`, `planned`),
§2.2 (Completion Protocol payload fields). Headings carry their canonical
file and section number so citations like `lifecycle §2.2` resolve
unambiguously. The canonical files are authoritative and hold the full
examples and word lists; read them when a condensed section is not
enough. When a canonical section changes, re-derive the condensed form
here (CI pins the canonical hashes via `sources.json`).

---

## conventions §1 — The Iron Law of grounding

```
Never write what you cannot cite or do not know.
```

Applies wherever an agent generates `executionRecord`, `decisions`, `description`, or `files`: claims must reference real code (paths that exist, functions that are defined), `description` must reflect actual scope, `files` must list paths observed or confirmed. When uncertain, write less. A short, true record is more valuable than a rich, fabricated one. `decisions` come from the conversation, not from artifact-mining.

---

## lifecycle §1 — Status lifecycle

```
draft → planned → in_progress → in_review → done
                                            cancelled (terminal, reachable from any non-terminal)
```

### Summary

| Status | Required fields | Forbidden fields | Trigger to leave |
|---|---|---|---|
| `draft` | `description`, `acceptanceCriteria` | `executionRecord`, `implementationPlan` | implementation plan saved → `planned` |
| `planned` | + `implementationPlan` (unabridged); all `depends_on` blockers `done` | `executionRecord` | someone claims via `piyaz_edit` (`set status='in_progress'`) → `in_progress` |
| `in_progress` | + active worker (one only) | — | work complete + record + ACs + Completion Protocol §2 run → `in_review` |
| `in_review` | + `executionRecord`, `decisions`, `files`, every AC evaluated, `prUrl` (optional sugar — when a PR was opened; backend upserts a `task_links` row with `kind='pull_request'`) | — | HOTL operator inspects PR and flips → `done` (or back to `in_progress` for rework) |
| `done` | (inherited from `in_review`) | — | terminal |
| `cancelled` | + `executionRecord` (rationale + what was tried), `decisions` | — | terminal |

### `draft`

Scope captured; the task is real but unbuilt. Cannot be coded directly. Transitions to `planned` when an implementation plan is written and saved on the task, unabridged; do not save summaries.

### `planned`

Implementation plan written, all `depends_on` blockers `done`, ready to claim. Transitions to `in_progress` when someone explicitly claims via `piyaz_edit` (`set status='in_progress'`), BEFORE starting work.

---

## lifecycle §2.2 — Populate the required fields (Completion Protocol)

`executionRecord`, `decisions`, `files`, `acceptanceCriteria`, plus `prUrl` when a PR was opened (backend upserts a `task_links` row with `kind='pull_request'` so the review subagent and detail UI can resolve the PR). The MCP server returns `_hints` if any are missing. Re-call with the additions before continuing.

For tasks that touched no repo files, pass `files=[]` explicitly; omitting the field leaves the prior value in place and the server's "missing files" hint will not clear.

(The implementer writes this payload once at `in_review` from its own extract; the planner does not pre-stage it in the plan.)

---

## artifacts §1 — Task artifact quality

### `description`

The first thing a coding agent or engineer reads when picking up a task; it must be enough on its own to start the work. Cover what the work is, who or what it serves, and where it sits in the architecture (for a bug: what is broken, when it manifests, the suspected cause). Include a solution sketch when you have one ("Use Drizzle, mirror the patterns in `lib/data/task.ts`" beats "Define the database tables"); do not pad with implementation guesses when the approach is uncertain.

Length: 2 to 4 sentences for most tasks, up to 6 to 8 for genuinely complex ones. Single-sentence descriptions are never acceptable; the server flags them in `_hints`.

```
GOOD:
"Build the habit completion endpoint at POST /api/habits/:id/complete. Inserts
into habit_logs with the user's timezone-adjusted date. Returns the updated
streak count. Idempotent on (habit_id, log_date): duplicate calls return the
existing log. Used by both the web dashboard and the iOS widget."

BAD: "Improve the database." / "Make auth better." / "Build the dashboard."
```

### `acceptanceCriteria`

2 to 4 items. Each criterion must be **binary**: a reviewer answers YES or NO without ambiguity.

```
GOOD: "Running bun run db:push creates all tables without errors"
GOOD: "spi_send returns within 50µs at 80MHz clock measured on logic analyzer"
BAD: "Database works" / "Tests pass" / "Performance is good"
```

Single-AC tasks are flagged by the server in `_hints`; rewrite them. Vague ACs ("works correctly", "performs well") must be rewritten before planning.

### `decisions`

One-liner per decision: **CHOICE + WHY**. Decisions come from the refinement, planning, or implementation conversation; record them without being asked, and never invent one that is not grounded in conversation, code, or the artifacts above.

```
GOOD: "Chose Redis for refresh tokens. Need fast revocation lookups."
BAD: "Used Drizzle" / "Decided to do it that way"
```

---

## artifacts §6 — Markdown formatting and tone

Applies to `description`, `acceptanceCriteria`, `executionRecord`, `implementationPlan`, `decisions`, and edge `note`; not to `files` (plain paths) or `tags` (kebab-case).

Structure: bullet lists for 3 or more items, backticks for code references, paragraph breaks between topics, headings only in long fields like `implementationPlan`. Tone: the text must read like an engineer wrote it, not a chatbot. No em dashes, no hedging or throat-clearing openers, no marketing words ("comprehensive", "robust", "leverage", "seamless"), no filler, no performative sign-offs. Subject-verb-object, active voice, concrete over abstract ("Adds 50ms p99" beats "improves performance"). Concision over padding, but clarity beats brevity: a genuinely complex task gets its 6 to 8 sentences. Full word lists and examples: canonical `artifacts.md` §6.
