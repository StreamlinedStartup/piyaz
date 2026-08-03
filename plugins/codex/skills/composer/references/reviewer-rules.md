# Reviewer rules (composer Phase 4 extract)

Condensed extract of the canonical piyaz references for the review agent.
Sources: `skills/piyaz/references/conventions.md` §1,
`skills/piyaz/references/lifecycle.md` §2.2, §2.3, §2.4, §3, and
`skills/piyaz/references/artifacts.md` §1 (`executionRecord`,
`decisions`), §6. Headings carry their canonical file and section number
so citations like `lifecycle §2.2` resolve unambiguously. The canonical
files are authoritative and hold the full examples and word lists; read
them when a condensed section is not enough. When a canonical section
changes, re-derive the condensed form here (CI pins the canonical hashes
via `sources.json`).

The reviewer verifies the Completion Protocol was honored; it does not
execute it. §2.2 and §2.3 below are what the implementer was required to
do; §3 is what the orchestrator runs after your verdict, fed by your
downstream-impact list.

---

## conventions §1 — The Iron Law of grounding

```
Never write what you cannot cite or do not know.
```

For the reviewer it applies to the verdict: every finding cites a real file path and line, every AC evaluation cites the diff or the executionRecord. When uncertain, write less. A short, true verdict is more valuable than a rich, fabricated one.

---

## lifecycle §2.2 — Populate the required fields

`executionRecord`, `decisions`, `files`, `acceptanceCriteria`, plus `prUrl` when a PR was opened (backend upserts a `task_links` row with `kind='pull_request'` so the review subagent and detail UI can resolve the PR). The MCP server returns `_hints` if any are missing.

For tasks that touched no repo files, `files=[]` is the correct positive answer to "what changed in the repo?", not the absence of an answer.

Non-code deliverables must be reviewable: committed in the PR when repo-resident, otherwise linked on the task or recorded in a `Deliverables` section of the `executionRecord` with the path or URL and the exact regeneration command. A claimed deliverable the reviewer cannot reach is a blocking finding.

## lifecycle §2.3 — Open a PR if the work changed code (what the implementer owed)

If `files` is non-empty AND the work was a real code change (not research, not decision-only, not Piyaz-only refinement), the implementer must have opened a PR:

- PR body follows the repo's PR template when one exists (`.github/PULL_REQUEST_TEMPLATE.md` and variants), the canonical concise default otherwise.
- The `taskRef` appears in `[BRACKETS]` (e.g. `[MYMR-83]`) exactly once, for the ONE primary task the PR builds. Bracket form triggers Piyaz PR-status tracking. Related tasks are referenced as plain links, no brackets.
- Summary maps from `executionRecord` (2 to 3 sentences); test plan maps from checked `acceptanceCriteria`; notes-for-reviewer maps from `decisions`.
- Sections are concise; empty optional sections beat fabricated content.

A missing PR on a code-changing task, a missing bracket ref, or a fabricated template section is a finding.

## lifecycle §2.4 — Skip the PR for these task types

A missing PR is legitimate (not a finding) for: research / investigation tasks, decision-only tasks, pure-Piyaz refinement tasks, tasks the user explicitly said "no PR" on, and data / BA work without a code repo (the artifact link or path belongs in `executionRecord` and `files` instead). When the data work IS in a git repo (a dbt project, a versioned SQL or notebook repo), the standard PR rules apply.

---

## lifecycle §3 — Propagate after every change (Iron Law)

```
A change that does not propagate did not happen.
```

The graph is Piyaz's value; skip propagation once and it lies. After any status change or significant refinement, the orchestrator (or the human) walks `piyaz_map view='neighbors'` and `view='downstream'` on the changed task and creates, updates, or removes edges whose notes or assumptions the change invalidated. The reviewer does not execute propagation; your downstream-impact list names the edges that will need attention.

---

## artifacts §1 — Task artifact quality

### `executionRecord` (only on `in_review`, `done`, and `cancelled`)

The implementer writes this field at the `in_review` transition; you verify it against the diff.

- **Length:** 3 to 5 sentences.
- **Distinct from `description`:** description = scope + role; executionRecord = HOW it was built (or WHY it was abandoned).
- **Include:** function names, file paths, endpoints, data formats.
- **Exclude:** debugging stories, false starts, filler.
- **For `cancelled`:** rationale (why abandoned), approaches tried, decisions learned.
- **Deliverables section (optional):** when the task ships non-code artifacts, a `## Deliverables` list (path or URL plus the exact regeneration command per artifact) extends the record beyond the sentence core.
- **Draft tasks must NOT carry an `executionRecord`.** That field implies the task shipped.

### `decisions`

One-liner per decision: **CHOICE + WHY**.

```
GOOD: "Chose Redis for refresh tokens. Need fast revocation lookups."
BAD: "Used Drizzle" / "Decided to do it that way"
```

An implementer `decisions` entry that is not grounded in the diff, the plan, or the conversation is a finding.

---

## artifacts §6 — Markdown formatting and tone

Applies to everything you write into the verdict. Structure: bullet lists for 3 or more items, backticks for code references, paragraph breaks between topics. Tone: the verdict must read like an engineer wrote it, not a chatbot. No em dashes, no hedging or throat-clearing openers, no marketing words ("comprehensive", "robust", "leverage", "seamless"), no filler, no performative sign-offs. Subject-verb-object, active voice, concrete over abstract ("Adds 50ms p99" beats "improves performance"). Concision over padding; the rule is "no fluff", not "no length". Full word lists and examples: canonical `artifacts.md` §6.
