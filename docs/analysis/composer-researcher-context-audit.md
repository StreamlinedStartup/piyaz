# Context audit: composer-researcher agent

Audit of `plugins/claude-code/agents/composer-researcher.md` (25.0 KB) and its
always-loaded extract `plugins/claude-code/skills/composer/references/researcher-rules.md`
(12.0 KB, pulled in via `@`-include) against two Anthropic guides for the
Claude 5 generation:

- "The new rules of context engineering for Claude 5-generation models"
- "A field guide to Claude Fable 5: Finding your unknowns"

Date: 2026-08-02. Combined payload per dispatch: ~37 KB, roughly 9-10K tokens,
loaded fresh on every composer Phase 1 run.

## Why this agent

The researcher is dispatched once per task in every composer run, so its prompt
is the most frequently paid context cost in the pipeline. It is also the agent
whose job description ("find what the task description does not say") maps most
directly onto the field guide's core concept of surfacing unknowns, which makes
it a good test of whether our prompt helps or hinders that.

## Summary verdict

The agent gets the hard things right: ownership boundaries, grounding, and
calibrated uncertainty are all first-class. Where it conflicts with the new
guidance is volume and rigidity. It restates the same constraint up to four
times, duplicates semantics the MCP tool schemas already carry, inlines
reference material that is rarely needed, and locks exploration into a
mandatory nine-step checklist. Applying the recommendations below would cut
the payload roughly in half without loosening any invariant that matters.

## What already matches the new rules

These sections should survive any trim untouched.

- **Calibrated unknowns surfacing.** The `open_questions` list, controlled
  `flags` vocabulary, numeric `confidence` with a 0.6 gate, and the
  `NEEDS_DECISION` status are exactly the field guide's "assume unknowns
  exist and route them to a human" pattern, implemented as an interface
  rather than as prose exhortation. This is the strongest part of the file.
- **Propose vs. apply gating** (`## Proposed rewrites`, the "same task or
  different task" litmus test at line 79). This is judgment-based framing,
  not a rigid rule: it gives the model a test to apply rather than an
  enumeration of cases. The context-engineering post holds this shape up as
  the target.
- **Code as reference.** Step 2 requires patterns cited as `path:line` and
  the brief's "Existing patterns to reuse" section forces source citations.
  The field guide is explicit that source code beats descriptions as a
  reference; the brief format bakes that in.
- **The Iron Law** (conventions §1). One sentence, universally applicable,
  judgment-compatible. Keep it inline.
- **Status ownership rules.** "You own zero transitions" protects a state
  machine shared across five agents and a human operator. The new guidance
  says cut rules that are wrong for some prompts; these are wrong for none,
  so they stay. They can still be stated once instead of twice (see below).
- **Structured return schema.** Attaching a schema at dispatch and letting
  validation enforce it is interface design in the sense the post means:
  the tool contract carries the constraint, not prose.

## Findings

### 1. Repetition instead of single placement

The post's rule 4: state each constraint in one place. The file violates this
internally several times over.

- The Fibonacci estimate bound (`1, 2, 3, 5, 8, 13`, never above 13) appears
  at lines 85, 144, and 240 of the agent file, and again in artifacts §2 of
  the extract. Four statements of one closed enum that the `piyaz_edit`
  schema already types.
- The `oversize-task` routing ("do not split it yourself, raise the flag")
  appears at lines 85, 144, and 209, and again in artifacts §5.
- Status prohibitions appear in `## Forbidden tools` (line 61) and then again
  as a dedicated five-bullet section (lines 65-73) enumerating each status
  value individually. The per-value bullets add nothing the one-line rule
  ("never include a status op; you own zero transitions") does not.
- The substantive-rewrite gate is specified in full at lines 77-83 and then
  restated per-field at lines 139-140.

Recommendation: one authoritative statement per constraint. Where a later
section needs it, point at the section instead of restating. Estimated
saving: 15-20% of the agent file.

### 2. Duplication of tool-schema and server-provided context

The post's rule 2 and 4 together: make the interface expressive and do not
repeat what the interface already says. Several passages restate content the
model receives anyway:

- The taskRef format section (conventions §4 in the extract, plus line 33 of
  the agent) duplicates the piyaz MCP server instructions, which every
  session already carries ("Refs are first-class... responses emit refs").
- Step 1's explanation of what `lens='agent'` contains duplicates the
  `piyaz_get` parameter description, which the model reads with the tool.
- The op-by-op enumeration of allowed `piyaz_edit` ops is worth keeping as
  policy (the server does not know composer phases), but the mechanics
  (what `str_replace` does, that `add` accretes) live in the tool schema.
  Keep the policy line ("refinement ops only, accretive, never `remove` or
  wholesale `set`"), drop the mechanics.
- The prose walkthrough of the structured-return fields (lines 233-249)
  partially restates the attached schema. Keep only the semantics the schema
  cannot express (that `estimate` drives downstream model selection, that
  DONE with 0/0 counts is rejected); drop field-by-field descriptions.

### 3. Everything upfront instead of progressive disclosure

The extract inlines all of its sections on every dispatch, but their usage
frequency differs sharply:

- artifacts §1 (description/AC/decisions rubric): used every run, in steps
  6-8. Justified inline.
- conventions §1 (Iron Law): one paragraph, used every run. Justified.
- artifacts §6 (tone rules, the banned-word list): only relevant when
  actually writing a description rewrite, and half of it duplicates the
  user-level style rules already in global context. Candidate for a
  Read-on-demand reference ("before rewriting a description, read
  `references/artifacts.md` §6").
- artifacts §2's domain example tables (embedded, aerospace, ML, financial
  tag examples): this repo is a web app; the cross-domain examples exist
  because the extract mirrors a generic canonical file. The researcher gets
  the project's actual tag vocabulary in its dispatch payload, which is the
  better reference anyway. Cut the tables, keep the three-dimension rule.
- artifacts §5 (granularity): two sentences of it matter to the researcher
  (the oversize threshold); the rest addresses the decompose agent.

The agent already has `Read` in its allowed tools, so deferring is free: keep
the per-run rubric inline, move the rest behind file pointers. Estimated
saving: half the extract, ~5-6 KB.

### 4. Extensive GOOD/BAD example blocks

The extract carries roughly 60 lines of few-shot examples (three GOOD
descriptions across domains, two GOOD AC blocks, three GOOD decisions, plus
BAD lists for each). The post warns that few-shot examples "constrain the
exploration space" for 5-generation models. These are content-quality
examples rather than tool-call examples, so they are less harmful than the
anti-pattern the post names, but the marginal example adds little: one GOOD
and one BAD per artifact type conveys the rubric. Cutting to that keeps the
signal at a third of the cost, and the cross-domain variants go with the
domain tables from finding 3.

### 5. Rigid procedure where judgment would do

"Run these in the order given; do not skip" (line 107) forces all nine steps
on every task. For a docs-only or chore task, step 3 (dependency version
research) and most of step 5 (NFR analysis) are wasted motion, and the field
guide's first anti-pattern is exactly this: when the prompt is too specific,
the model follows it even when pivoting would be better. The steps themselves
are a good coverage map; the fix is reframing from mandate to checklist of
concerns: "cover each area below that the task plausibly touches; state
`n/a` in the brief for areas you skipped and why." The brief format already
forces every section to appear, so skipped areas remain visible to the
orchestrator; nothing is lost except forced busywork.

The same applies to micro-prescriptions like "gh pr list --state merged
--limit 5, then gh pr view on the two most recent" (line 125). State the
goal (extract the commit convention from recent merged PRs) and let the
model pick the commands.

### 6. The merged mandate doubles the file's complexity

Lines 91-103 turn the researcher into a conditional researcher-plus-planner,
importing a summary of the planner's rubric, its failure routing, and an
extended schema. Every non-merged dispatch still pays for all of it, and the
mode driven by dispatch text ("an explicit authority grant lifts the
restrictions above") is the opposite of interface design: the same prompt
must hold two contradictory permission sets and switch on prose. If the
merged path is the dominant composer path, the cleaner structure is a
separate `composer-researcher-planner.md` agent (or building the merged
prompt at dispatch time from the two rule extracts), so each file states one
permission set unconditionally. That also removes the most dangerous
sentence in the file, "Only that grant lifts the restrictions", which asks
the model to overrule its own explicit prohibitions based on who asked.

### 7. Field-guide alignment on unknown unknowns is implicit, not framed

The researcher's job is a blind spot pass over the task, and steps 2-5
operationalize it well, but nothing invites the model to look outside the
checklist. The field guide's core claim is that Fable-generation output
quality is bottlenecked by surfacing unknown unknowns, and that the model is
good at finding them when asked directly. One sentence would capture this:
after the coverage areas, "separately, note anything about this task that
the checklist did not ask about but that would change the plan if the
planner knew it." The `open_questions` channel already exists to carry the
answer.

### 8. Model pin

The frontmatter pins `model: sonnet`. The context-engineering post's headline
result (80% prompt reduction, no eval drop) was measured on 5-generation
models, which current Sonnet is, so the trims above apply. If the pin is ever
lowered to Haiku for cost, revisit: smaller models are the audience the old
heavy-constraint style was built for, and some of the scaffolding cut here
would earn its place back.

## Recommended change list, in order of value

1. Deduplicate the estimate bound, oversize routing, status prohibitions,
   and rewrite gate to one statement each (finding 1).
2. Split the merged mandate into its own agent file or dispatch-time prompt
   assembly (finding 6).
3. Slim the extract: keep conventions §1 and artifacts §1 (with one GOOD and
   one BAD example each), move §2 tables, §5, and §6 behind Read-on-demand
   pointers (findings 3, 4).
4. Reframe the procedure as coverage areas with an explicit skip convention,
   and drop command-level prescriptions (finding 5).
5. Cut schema restatement and tool-doc duplication down to the semantics the
   schema cannot carry (finding 2).
6. Add the one-line unknown-unknowns invitation (finding 7).

Estimated result: ~18-20 KB combined payload instead of 37 KB, with every
load-bearing invariant (status ownership, grounding, accretive edits,
rewrite gating, confidence gating) intact.

## Implementation status (2026-08-02)

Change-list items 1 and 3-6 landed on branch `refactor/agent-context-trim`
(commits `fb46f5a`, `268a2be`, `0c13646`, `3308804`, `0ac3844`), extended
to the planner, implementer, review, and onboarding agents and their
extracts: 18 files, +212/-1221 lines, `check:plugins` green. Item 2 (the
merged-mandate split) is deferred to task PYZ-1; the same treatment for
the remaining agents (brainstorm, decompose family, manage) is PYZ-2.

## Follow-ups if the changes land

- `researcher-rules.md` mirrors canonical files under
  `skills/piyaz/references/`; per its own header, edits go to both. Slimming
  the extract does not require touching the canonical files, since the
  extract becomes pointers plus the two kept sections.
- The other three composer extracts (implementer 14.4 KB, planner 9.8 KB,
  reviewer 7.3 KB) and the larger agents (`review.md` at 41 KB,
  `onboarding.md` at 34.7 KB) very likely show the same patterns and should
  get the same audit before a wholesale rewrite pass.
- `bun run sync:plugins` must run after any edit to the shared skills, per
  CLAUDE.md.
