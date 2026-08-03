# Shared research procedure (composer Phase 1)

Loaded by `agents/composer-researcher.md` (research-only dispatch) and
`agents/composer-research-planner.md` (the workflow's merged
research+plan phase). Permission sets live in the loading agent's own
file; this file holds the procedure, the brief format, and the STATUS
semantics they share. Citations (`conventions §1`, `artifacts §1`)
resolve inside the researcher-rules extract loaded alongside it.

## Substantive rewrites: propose, do not apply

Refinements to scalar fields (`description`, `category`, `priority`, `estimate`) overwrite, not append. Most refinements are *sharpening*: same scope, sharper wording. Apply those silently. A *substantive rewrite* changes what the task IS, not how it is described.

Litmus test: would a reasonable user reading the original description vs the proposed one say "same task" or "different task"? If different, you are proposing a rewrite, not a refinement.

For substantive rewrites, do not apply. Emit the proposed value in the brief's `## Proposed rewrites` section (one entry per field with a one-line rationale) and continue with the rest of the brief. The orchestrator gates the rewrite with the user before advancing to planning. On accept, the orchestrator applies the rewrite and re-dispatches a fresh research run on the rewritten task, so planning is grounded in the post-rewrite scope. On deny, the iteration ends.

Small refinements (one-line clarification, AC binary-rewrite where intent was clear, tag dimension fill-in, estimate refinement within `1, 2, 3, 5, 8, 13`, category correction to a project-defined value, priority refinement) apply directly. The HOTL gate exists for scope changes, not for tightening prose.

## Procedure

Cover each area below that the task plausibly touches; where one clearly does not apply (a docs-only task needs no dependency-version research), skip it and say so in the brief's matching section rather than leaving it silent. Steps 2–5 can fan out in parallel where they do not depend on each other. Separately from this checklist: anything you notice that the checklist did not ask about but that would change the plan if the planner knew it belongs in the brief or in `open_questions`.

1. **Read the task.** One fetch: `piyaz_get lens='agent' task='<taskRef>'`. Do not also fetch `lens='working'` — it is ~80% duplicate of the agent bundle. When wider 1-hop sibling context matters, add `piyaz_map view='neighbors' task='<taskRef>'` instead. Note any ambiguous criteria or thin descriptions; you flag these for planning to refine.

2. **Map the task to the codebase.** Identify:
   - Files the implementer will touch (use `Glob` + `Grep` against the task's description, category, and tag dimensions).
   - Existing patterns or abstractions the implementer should reuse (search by intent, not by name; e.g. for an auth task, grep for existing middleware patterns).
   - Tests that cover the touched files (look for `.test.`, `.spec.`, `__tests__/` siblings).
   - Sibling tasks that already shipped adjacent work (`piyaz_search` by tag or title fragment; read their `executionRecord` for context).

3. **Investigate external dependencies.** For any library, framework, SDK, or API the task touches:
   - Read the project's pinned version (`package.json`, `requirements.txt`, `Cargo.toml`, `go.mod`).
   - Resolve current docs via `context7` (preferred) or `WebSearch` (fallback). Cite the doc URL or context7 library id.
   - Flag version drift when the pin is more than one minor behind current and the task's implementation depends on a newer API.

4. **Audit project conventions.** Read these sources, in order:
   - `CLAUDE.md` at the project root and any nested `CLAUDE.md` files (use `Grep` to locate). House rules live here.
   - Lint and format configs: `eslint.config.*`, `biome.json`, `ruff.toml`, `.prettierrc*`, `package.json` scripts.
   - Recent merged PRs (`gh pr list` / `gh pr view`) for commit-format conventions.
   - PR template: `.github/pull_request_template.md` (and lowercase/path variants).
   - Extract: commit-message convention, test command, typecheck command, lint command, PR template path. The implementer reads these from your brief and matches house style verbatim.

5. **Reason about non-functional requirements.** For the work the task implies, identify:
   - **Security**: input validation boundaries, authn/authz checks, secret handling, SQL/command injection surfaces. Cite the project's existing security patterns where they exist; flag where the task crosses a trust boundary without an established pattern.
   - **Performance**: latency-sensitive paths, expected throughput, data volumes. Cite measured baselines if they exist; flag missing instrumentation otherwise.
   - **Reliability**: failure modes the implementer must handle vs. ones to let propagate, retry semantics, idempotency requirements.
   - **Observability**: log/metric/trace expectations consistent with the rest of the codebase.

6. **Score acceptance criteria.** Walk the target's current `acceptanceCriteria` and score each against the binary-AC rubric in artifacts §1. Apply binary rewrites for ambiguous criteria via `piyaz_edit` with `{op:'update', collection:'acceptanceCriteria', id:'<id>', text:'<rewrite>'}`. Criterion ids are visible in your context bundle — each rendered criterion line carries its backticked id (or fetch `fields=['acceptanceCriteria']`); use those ids, never invent one. Missing coverage gets an `{op:'add', collection:'acceptanceCriteria', text:'...'}` op. Quantity bounds live in artifacts §1; do not restate them, just hit them.

7. **Apply refinements.** Fold your findings back into the target task with one `piyaz_edit` call carrying the ordered ops (atomic; split only when over the 20-op cap). The fields you may touch are the refinement fields in *Allowed tools*; each must be backed by a citation you would put in the brief. Per-field rules:

   - **`description`**: when the existing description fails the rubric in artifacts §1, rewrite it, citing the codebase reads that justify the rewrite. Scope-preserving sharpening applies directly; a scope change goes to `## Proposed rewrites` (see *Substantive rewrites* above).
   - **`acceptanceCriteria`**: apply the binary rewrites/additions from step 6 directly. A change to the AC composition itself (different criteria, different coverage scope) goes to `## Proposed rewrites`.
   - **`tags`**: bring every task to the full three-dimension shape before handoff: exactly 1 work-type, at least 1 cross-cutting concern, at most 2 tech. This is a gate, not optional fill-in. A task that reaches planning with a missing or degenerate dimension is a research miss; you own `tags`, so no later phase can fix it. Strip any `area:` prefix: codebase area is `category`'s job, never a tag (artifacts §2). Map an `area:x` tag to the matching category, or drop it. Run `piyaz_get view='meta'` first to reuse existing vocabulary.
   - **`category`**: set to the closest match from `piyaz_get view='meta'`. Never coin a new category, and never use process phases (`requirements`, `planning`, `review`), work types, or priorities as a category — those shapes are forbidden; categories are subsystems/product areas only.
   - **`priority`**: adjust when your investigation surfaces evidence the current value is wrong (e.g., a security boundary the task crosses argues for `core` or `urgent`).
   - **`estimate`**: adjust within the Fibonacci scale when scope drift is evident, never above `13`. If the true scope exceeds what `13` represents, raise `oversize-task` in *Flags* so the orchestrator routes to `piyaz:decompose-task`; splitting the task is the decompose agent's job, not yours. No `decisions` entry for the bump; the audit log records it.
   - **`decisions`**: append a one-liner only when refinement work produced a real CHOICE + WHY (see artifacts §1 for shape and examples). Real cases: picking one library version or pattern over an alternative when the codebase or docs argue for it; choosing to reuse an existing module rather than introducing a new one. Findings, measurements, and pinned-version facts are *not* decisions; those belong in the brief's *Security/performance/...* and *External dependencies* sections, not in `decisions`. Better an empty `decisions` list than fabricated entries.

   When in doubt, leave the field alone and surface the call in `open_questions`. Speculation in a `description` rewrite is worse than a thin description.

8. **Self-verify before returning.** Research is the foundation; a refinement mistake here cascades into a wrong plan and wrong code, wasting every downstream phase. Before you return, re-read the refined task (`piyaz_get lens='planning' task='<taskRef>'`) and check each item:

   - Every acceptance criterion is **binary**: a reviewer answers YES or NO without judgement (artifacts §1). An ambiguous criterion that survived to your return is a defect. Rewrite it; if you cannot, flag `ambiguous-criterion-unresolved` and lower confidence.
   - Every path in *Files to touch* exists in the repo or is explicitly a new file the work creates. Drop or correct any path you cannot confirm.
   - The refined `description` matches what the codebase actually supports: no scope you invented, no API you did not verify against docs or source.
   - Every refinement you applied is backed by a citation you can put in the brief. A refinement without a citation is ungrounded; revert it.

   Any check that fails and that you cannot fix lowers your confidence honestly and adds the matching flag. A calibrated confidence below 0.6 gates the task to the user; passing shaky research through as confident is the failure this step exists to prevent.

9. **Surface open questions.** Anything you cannot cite, any ambiguity that the refinements did not resolve, any decision that needs the user's input (which library to use, which behavior is correct, etc.) goes in `open_questions`. The orchestrator surfaces these before advancing to planning.

## Output format

Return one markdown brief with the following exact sections in this order. Do not omit any section; use `none` when a section has no content. No preamble, no postscript.

```markdown
# Research brief: <taskRef>

## Files to touch
- `<repo-relative path>`: `<one-sentence reason citing the task's description or a specific upstream decision>`
- ...

## Existing patterns to reuse
- `<pattern name>`: `<example path : line range>`. `<one-sentence why it applies>`.
- ...

## External dependencies and versions
- `<library>@<pinned-version>`; current `<current-version>`; citation: `<context7 library id or doc URL>`; drift: `<none | minor | major>`; notes: `<one sentence>`
- ...

## Project conventions
- Commit format: `<convention>`; citation: `<file path or PR number>`
- Test command: `<command>`; citation: `<file path>`
- Typecheck command: `<command>`; citation: `<file path>`
- Lint command: `<command>`; citation: `<file path>`
- PR template: `<path or "none">`

## Security, performance, reliability, observability
- Security: `<paragraph; cite existing patterns>`
- Performance: `<paragraph; cite baselines or flag absence>`
- Reliability: `<paragraph>`
- Observability: `<paragraph>`

## Applied refinements
- `<field>`: `<one-sentence summary of what you changed and why>`; citation: `<file:lines | url | piyaz taskRef>`
- ...

(use `none` when no refinements were warranted)

## Proposed rewrites
- `<field>` (`description` or `acceptanceCriteria`): `<proposed value verbatim>`; rationale: `<one sentence>`; citation: `<file:lines | url | piyaz taskRef>`
- ...

(use `none` when no substantive rewrites were proposed; sharpening refinements go to *Applied refinements* above, not here)

## Open questions
- `<one sentence per question>`
- ...

## Flags
- `<flag>` from the controlled vocabulary: `oversize-task` (true scope exceeds what `13` represents; route to decompose), `missing-citation`, `dep-mismatch`, `ambiguous-criterion-unresolved`, `version-drift-major`, `security-boundary-uncovered`, `external-input-required`
- ...

## Confidence
<number in [0,1]; your overall confidence the refinements and findings are accurate and complete. Below 0.6 means the orchestrator should surface open questions to the user before planning.>

STATUS: <DONE | DONE_WITH_CONCERNS | NEEDS_DECISION | BLOCKED> — <one-line reason>
```

## Choosing STATUS

The STATUS line is the last line of your return and the only thing the orchestrator branches on. Pick exactly one:

- `NEEDS_DECISION`: any of — you raised `oversize-task`, your `## Proposed rewrites` section is non-empty, your confidence is below 0.6, or you raised `external-input-required`. The reason line names which trigger fired.
- `BLOCKED`: you could not ground your findings at all (repo unreadable, task unresolvable, Piyaz unreachable).
- `DONE_WITH_CONCERNS`: brief is complete and nothing gates, but you raised non-gating flags (`version-drift-major`, `security-boundary-uncovered`, `missing-citation`, `dep-mismatch`, `ambiguous-criterion-unresolved`).
- `DONE`: brief complete, no flags, confidence ≥ 0.6, no proposed rewrites.

The brief feeds the planning half (the Phase 2 planner on a research-only dispatch; your own plan half on a merged dispatch). Keep it scannable: it is read once and acted on; a wall of prose buries the actionable parts. The refinements you applied are already in Piyaz and are read back via `piyaz_get lens='planning'`; the brief is the *findings* the plan is written against.

## Knowledge write-back

Before returning: if the research surfaced a durable, reusable finding (a library version quirk verified against docs, a house convention you had to reconstruct from the codebase, a constraint no note records), write it via `piyaz_note create` as a `knowledge` note with a one-line `summary`; check `piyaz_note list` first and reuse existing folders. Iron Law: cite the doc URL, manifest line, or file that grounds it. Task-specific findings belong in the brief and the task's refinement fields, not in notes; write a note only for what outlives this task. Never create `guidance` notes or set `feedMode`.
