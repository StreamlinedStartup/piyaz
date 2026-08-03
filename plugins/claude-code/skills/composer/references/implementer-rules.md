# Implementer rules (composer Phase 3 extract)

Condensed extract of the canonical piyaz references for the composer
implementer. Sources: `skills/piyaz/references/conventions.md` §1, §2,
`skills/piyaz/references/lifecycle.md` §1 (Summary, `in_progress`,
`in_review`), §2 (entire Completion Protocol, 2.1–2.4), and
`skills/piyaz/references/artifacts.md` §1 (`executionRecord`,
`decisions`, `files`), §6. Headings carry their canonical file and
section number so citations like `lifecycle §2` resolve unambiguously.
The canonical files are authoritative and hold the full examples and
word lists; read them when a condensed section is not enough. When a
canonical section changes, re-derive the condensed form here (CI pins
the canonical hashes via `sources.json`).

---

## conventions §1 — The Iron Law of grounding

```
Never write what you cannot cite or do not know.
```

Applies wherever an agent generates `executionRecord`, `decisions`, `description`, or `files`: claims must reference real code (paths that exist, functions that are defined, endpoints that are routed, commits that are in the log), and `files` must list paths modified, observed, or confirmed. When uncertain, write less. A short, true record is more valuable than a rich, fabricated one. `decisions` come from the conversation and the work, not from artifact-mining; never invent them.

---

## conventions §2 — Tool descriptions and `_hints` are runtime instructions

Every Piyaz tool injects two things into your context at use time: the tool's description and parameter schema before the call, and a `_hints` array in the response after it. These are server-side rules and state you cannot see otherwise; they override any prior plan you had. **Read on every tool call. Act before continuing.**

When multiple `_hints` fire in one response, service required-field hints first (the task is not in its final state until they clear), then informational follow-ups (propagation, suggested next call). Skipping a hint is operating on stale information.

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

### `in_progress`

Active implementation, exactly one engineer or agent working it. Should not span sessions: if work pauses, leave a note in the task or move it back to `planned`. Transitions to `in_review` when the work is complete, the payload fields are populated, ACs are evaluated, and the Completion Protocol (§2) has run.

### `in_review`

Work finished, PR open, full Completion Protocol payload populated, checks green, awaiting human review. Cannot be self-promoted to `done` by any agent; the HOTL operator owns `in_review → done` (no additional payload needed) and may flip back to `in_progress` for rework.

---

## lifecycle §2 — Completion Protocol

Before transitioning a task to `in_review`, `done`, or `cancelled`:

### 2.1. Detect mode by transcript

- **Dispatched mode**: your context shows you were invoked via the Task tool by a parent agent. Mark `in_review` directly with the full payload; the HOTL operator finalizes to `done`. Return to the parent with the task ref and a one-sentence summary. Do not ask.
- **Direct mode**: invoked by the user in a normal session. Ask "Ready to mark this `in_review`?" with a one-sentence executionRecord preview and wait for explicit confirmation.
- **Uncertain**: default to asking. A spurious confirmation prompt is cheap; an unauthorized status change is expensive.

### 2.2. Populate the required fields

One `piyaz_edit` call carries the whole payload as ordered ops: `set executionRecord`, one `add` per decision, `set files`, `check`/`uncheck` each acceptance criterion by its id, `set prUrl` when a PR was opened (backend upserts a `task_links` row with `kind='pull_request'` so the review subagent and detail UI can resolve the PR), and the `set status` transition. The call is atomic; the MCP server returns `_hints` if anything is missing. Re-call with the additions before continuing.

For pure spec-review / docs / decision-only / Piyaz-only refinement tasks that touched no repo files, `set files` with `value=[]` explicitly. Omitting the op leaves the prior value in place and the server's "missing files" hint will not clear. The empty array is the correct positive answer to "what changed in the repo?", not the absence of an answer.

Criterion ids come from `piyaz_get lens='working'` or `fields=['acceptanceCriteria']`; evaluate each against the actual work. A fix or rework rotation re-`set`s the author's own `executionRecord` to the folded final shipped state instead of appending per-rotation narrative.

Non-code deliverables (a generated report, data file, rendered doc, dataset, benchmark result, dashboard) must be reviewable: commit repo-resident artifacts in the PR; otherwise link them on the task or record the path or URL plus the exact regeneration command in a `Deliverables` section of the `executionRecord`. Agent worktrees are ephemeral; an uncommitted, unlinked output is gone by review time.

### 2.3. Open a PR if the work changed code

If `files` is non-empty AND the work was a real code change (not research, not decision-only, not Piyaz-only refinement):

**Detect a PR template** in the repo at one of these paths (or similar):

- `.github/PULL_REQUEST_TEMPLATE.md`
- `.github/pull_request_template.md`
- `.github/PULL_REQUEST_TEMPLATE/<name>.md`
- `docs/pull_request_template.md`

**If a template exists**: fill it. Map task fields onto template sections only where they fit. Leave a section blank rather than invent content. Common mappings:

- Linked issue / linked task: include the `taskRef` in `[BRACKETS]` (e.g. `[MYMR-83]`). Bracket form triggers Piyaz PR-status tracking; use it for the ONE primary task this PR builds. Reference any related tasks elsewhere as plain links (no brackets). Add `Closes #N` on its own line if a GitHub issue is being resolved.
- Summary section: 2 to 3 sentences from `executionRecord`.
- Test plan / verification section: the `acceptanceCriteria` items that are checked.
- Decisions or notes-for-reviewer section if present: relevant entries from `decisions`.

**If no template exists**: use this concise default.

```markdown
## Summary

**Task Reference**: [MYMR-XXX]
<!-- The ONE primary task this PR builds. Brackets trigger Piyaz
     PR-status tracking. Use them only here. Reference any related
     tasks elsewhere as plain links (no brackets). -->

<!-- What does this PR change and why? If it resolves a GitHub issue,
     add "Closes #N" on its own line. -->

## Type of change

- [ ] Bug fix
- [ ] New feature
- [ ] Refactor / cleanup
- [ ] Documentation

## Testing

- [ ] Tested locally with `<command>`
- [ ] Linting and formatting pass (`<command>`)
- [ ] Type or build check passes (`<command>`)

## Notes for reviewer

<!-- Anything non-obvious: tradeoffs, follow-up work, alternatives
     considered. Skip if there is nothing useful to add. -->
```

Open the PR with `gh pr create --title '<task title>' --body "$(cat <<'EOF' ... EOF)"`.

**Always concise.** Do not pad sections to look thorough. Empty optional sections beat fabricated content. If the template has prompt questions you cannot answer, skip them rather than make answers up.

### 2.4. Skip the PR for these task types

- Research / investigation tasks (no code change).
- Decision-only tasks.
- Pure-Piyaz refinement tasks (no repo changes).
- Tasks the user explicitly said "no PR" on.
- Data and BA work without a code repo (dashboard tweaks, workbooks, metric sign-offs, ad-hoc SQL attached to a ticket). The deliverable lives outside git; record the artifact link or path in `executionRecord` and `files` instead of opening a PR. When the data work IS in a git repo (a dbt project, a versioned SQL or notebook repo), open a PR per the standard rules above.

When in doubt, ask the user before opening.

---

## artifacts §1 — Task artifact quality

### `executionRecord` (only on `in_review`, `done`, and `cancelled`)

You write this field at the `in_review` transition; it is the core of your Completion Protocol payload.

- **Length:** 3 to 5 sentences.
- **Distinct from `description`:** description = scope + role; executionRecord = HOW it was built (or WHY it was abandoned).
- **Include:** function names, file paths, endpoints, data formats.
- **Exclude:** debugging stories, false starts, filler.
- **For `cancelled`:** rationale (why abandoned), approaches tried, decisions learned.
- **Deliverables section (optional):** when the task ships non-code artifacts, a `## Deliverables` list (path or URL plus the exact regeneration command per artifact) extends the record beyond the sentence core.
- **Draft tasks must NOT carry an `executionRecord`.** That field implies the task shipped.

### `decisions`

One-liner per decision: **CHOICE + WHY**. When a choice is settled (by you against the codebase, or with the user), record it without being asked; never invent one that is not grounded in conversation, code, or the artifacts above.

```
GOOD: "Chose Redis for refresh tokens. Need fast revocation lookups."
BAD: "Used Drizzle" / "Decided to do it that way"
```

### `files`

Plain repo-relative path strings, no backticks or quoting; every file created or modified. Empty `files=[]` is the correct value whenever paths cannot be cited (research or decision-only tasks, Piyaz-only refinements). Leave empty rather than speculate.

---

## artifacts §6 — Markdown formatting and tone

Applies to `description`, `acceptanceCriteria`, `executionRecord`, `implementationPlan`, `decisions`, and edge `note`; not to `files` (plain paths) or `tags` (kebab-case).

Structure: bullet lists for 3 or more items, backticks for code references, paragraph breaks between topics, headings only in long fields like `implementationPlan` and the executionRecord's optional `Deliverables` section. Tone: the text must read like an engineer wrote it, not a chatbot. No em dashes, no hedging or throat-clearing openers, no marketing words ("comprehensive", "robust", "leverage", "seamless"), no filler, no performative sign-offs. Subject-verb-object, active voice, concrete over abstract ("Adds 50ms p99" beats "improves performance"). Concision over padding, but clarity beats brevity; the rule is "no fluff", not "no length". Full word lists and examples: canonical `artifacts.md` §6.
