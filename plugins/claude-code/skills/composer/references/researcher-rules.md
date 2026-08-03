# Researcher rules (composer Phase 1 extract)

Condensed extract of the canonical piyaz references for the composer
researcher. Sources: `skills/piyaz/references/conventions.md` §1, §4 and
`skills/piyaz/references/artifacts.md` §1 (`description`,
`acceptanceCriteria`, `decisions`), §2, §5, §6. Headings carry their
canonical file and section number so citations like `conventions §1`
resolve unambiguously. The canonical files are authoritative and hold the
full examples and word lists; read them when a condensed section is not
enough. When a canonical section changes, re-derive the condensed form
here (CI pins the canonical hashes via `sources.json`).

---

## conventions §1 — The Iron Law of grounding

```
Never write what you cannot cite or do not know.
```

Applies wherever an agent generates `executionRecord`, `decisions`, `description`, or `files`: claims must reference real code (paths that exist, functions that are defined), `description` must reflect actual scope, `files` must list paths observed or confirmed. When uncertain, write less. A short, true record is more valuable than a rich, fabricated one. `decisions` come from the conversation, not from artifact-mining.

---

## conventions §4 — taskRef format

Tool responses include a `taskRef` like `WHL-214`. Refs are first-class everywhere: use them in user-facing output AND in tool calls (`task='WHL-214'`, `project='WHL'`); UUIDs are the fallback when a ref is ambiguous across teams. Chain the refs that responses emit; never invent one.

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

## artifacts §2 — Tag dimensions and first-class fields

Every task, in every status, carries tags across three dimensions. Reuse existing tags from `piyaz_get view='meta'` before coining new ones.

| Dimension | Count | Vocabulary |
|---|---|---|
| **Work type** | exactly 1 | `bug`, `feature`, `refactor`, `docs`, `test`, `chore`, `perf` |
| **Cross-cutting concern** | ≥1 | quality attribute (`security`, `a11y`, `dx`, `perf`, `reliability`, `observability`, `i18n`, `compliance`, `safety`) or a feature cluster spanning multiple categories (`onboarding-flow`, `agent-loop`) |
| **Tech** | at most 2 | most important stack pieces the task touches; pull from the project's actual manifest deps, never invent |

`priority` (`urgent`, `core`, `normal`, `backlog`), `estimate` (Fibonacci: `1`, `2`, `3`, `5`, `8`, `13`; larger than `13` means split), and `assigneeIds` are first-class fields, NOT tags.

**Do NOT tag:** priority (the field's job), status (the field's job), generic adjectives, or codebase area. Area is `category`'s job; the test is "would this name plausibly be a category in some other project shape?" (`auth`, `payments`, `render-loop` all answer YES). Coining an area-shaped tag because the category list lacks a slot is a category-list bug, not a tag. If the user explicitly tagged something, preserve their tags and add missing dimensions.

---

## artifacts §5 — Granularity

**1 to 4 hours per task**; a coding agent completes one in a single session. When in doubt, split. Splitting is the decompose agent's job; the researcher's part is raising `oversize-task` when the true scope exceeds what `13` represents.

---

## artifacts §6 — Markdown formatting and tone

Applies to `description`, `acceptanceCriteria`, `executionRecord`, `implementationPlan`, `decisions`, and edge `note`; not to `files` (plain paths) or `tags` (kebab-case).

Structure: bullet lists for 3 or more items, backticks for code references, paragraph breaks between topics, headings only in long fields. Tone: the text must read like an engineer wrote it, not a chatbot. No em dashes, no hedging or throat-clearing openers, no marketing words ("comprehensive", "robust", "leverage", "seamless"), no filler, no performative sign-offs. Subject-verb-object, active voice, concrete over abstract ("Adds 50ms p99" beats "improves performance"). Concision over padding, but clarity beats brevity: a genuinely complex task gets its 6 to 8 sentences. Full word lists and examples: canonical `artifacts.md` §6.
