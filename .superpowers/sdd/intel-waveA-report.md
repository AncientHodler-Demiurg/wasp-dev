# Intel Wave A Report

**Status:** DONE
**Branch:** multispec-foundation
**Date:** 2026-06-20

## Changes Made

### File 1: `plugins/bee/skills/guide/SKILL.md`

**(a) Merge-back routing added:**
- Section 1 Multi-Phase Intelligence table: new row for "Promoted spec finished (phases COMMITTED, location is a worktree)" → `/bee:workspace complete spec-<slug>` THEN `/bee:complete-spec`, with rationale about stranded code.
- Section 2 Build table: new row "Merge a finished promoted spec back in-place" → `/bee:workspace complete spec-<slug>`.
- Section 3 Workflow Efficiency: replaced the bare `IF spec complete: suggest /bee:complete-spec` line with a branching rule — worktree path first, then in-place path. Includes the explicit NEVER rule for sending a promoted spec straight to /bee:complete-spec.

**(b) Per-spec memory routing added:**
- Section 3: new "Memory Capture" subsection (before Idea Capture) — triggers offer of `/bee:memory` → spec memory when a constraint is scoped to the current feature; redirects to global user.md for project-wide rules.
- Section 2 Session Management: `/bee:memory` row reworded from "View preferences" to "View/manage memory — global user.md + per-spec memory.md".

**(c) Queue / plan-ahead routing added:**
- Section 1 Multi-Phase Intelligence table: new row for "Start a 2nd feature while one is in progress" → `/bee:new-spec` with note that it queues rather than archives.
- Section 2 Start Something: new row "Start another feature while one is still building" → `/bee:new-spec (queues alongside; does NOT archive the active spec)".
- Section 3 Workflow Efficiency: new bullet for the queue case — `/bee:new-spec` queues a 2nd spec, switch with `/bee:spec use`, or promote for parallel build.

**(d) Dashboard trigger broadened:**
- Section 3 Workflow Efficiency: replaced the literal "what are my specs" match with a broader list of status/overview triggers ("where am I", "what's active", "ce spec-uri am"). Added distinction: `/bee:spec list` for quick roster vs `/bee:spec dashboard` when a worktree is present (topology view).

**(e) NO_SPEC multi-spec note added:**
- Section 1 Spec-Level States: one-line note added immediately below the table explaining that NO_SPEC reflects the focused spec only; if the registry has active specs, suggest `/bee:spec use` / `/bee:spec list` rather than assuming the project is idle.

### File 2: `plugins/bee/commands/complete-spec.md`

**Worktree advisory added (Step 0.5):**
- Inserted a new Step 0.5 between Step 0 (resolve target spec) and Step 1 (validation guards).
- Queries the registry via `specs-cli.js list --active --json`, reads the resolved slug's `location` field.
- If location is not `in-place`: displays an advisory explaining the stranded-code risk, then presents an AskUserQuestion with "Proceed anyway (already merged back)" vs "Cancel — run workspace complete first".
- Cancel path prints the corrective action and stops.
- If location is `in-place` or the spec is absent from the registry (legacy spec), the step is skipped transparently.
- Tone matches sibling commands (progress.md, spec.md) — advisory, not hard-block.

## Test Results

- Full suite: 183/183 pass, 0 fail (identical to pre-change baseline).
- `bee-guide-skill.test.js`: 41/41 pass.
- `complete-spec-command.test.js`: 62/62 pass.

## Concerns

None. Both files are prose/docs only; no runtime logic was changed. The worktree advisory in complete-spec.md follows the soft-block pattern already established in the codebase and the test count increase (3 more AskUserQuestion calls, now 6 total, which exceeds the ≥3 assertion) is fine.
