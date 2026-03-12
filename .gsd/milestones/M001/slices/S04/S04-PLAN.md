# S04: Remove git commands from prompts

**Goal:** No raw git commands in LLM-facing prompts (except worktree-merge.md for conflict resolution).
**Demo:** `grep -rn 'git add\|git commit\|git update-index\|git hash-object' src/resources/extensions/gsd/prompts/ --include="*.md" | grep -v worktree-merge.md` returns zero results.

## Must-Haves

- execute-task.md step 16 replaced — no `git add`, `git commit`, `git update-index`, or `git hash-object`
- complete-slice.md step 10 replaced — no `git add` or `git commit`; preserves "do not squash-merge manually" context
- replan-slice.md step 6 replaced — no `git add` or `git commit`
- complete-milestone.md step 9 replaced — no `git add` or `git commit`
- worktree-merge.md untouched
- Step numbering remains sequential in all modified files
- `npm run build` passes
- `npm run test` passes

## Proof Level

- This slice proves: contract
- Real runtime required: no
- Human/UAT required: no

## Verification

- `grep -rn 'git add\|git commit\|git update-index\|git hash-object' src/resources/extensions/gsd/prompts/ --include="*.md" | grep -v worktree-merge.md` returns zero lines
- `npm run build` passes
- `npm run test` passes
- worktree-merge.md unchanged (diff shows no modifications)

## Observability / Diagnostics

Not applicable — this slice modifies static prompt templates with no runtime behavior.

- Runtime signals: none
- Inspection surfaces: grep across prompt files
- Failure visibility: none
- Redaction constraints: none

## Integration Closure

- Upstream surfaces consumed: GitService auto-commit mechanism wired in S02 (auto.ts lines 498–504) — makes manual commit instructions in prompts redundant
- New wiring introduced in this slice: none
- What remains before the milestone is truly usable end-to-end: S05 (merge guards, snapshots, auto-push, rich commits), S06 (cleanup and archive)

## Tasks

- [x] **T01: Replace raw git commands in all four prompt files** `est:15m`
  - Why: R011 — LLMs should not run git commands; the GitService auto-commit (wired in S02) makes these instructions redundant and harmful
  - Files: `src/resources/extensions/gsd/prompts/execute-task.md`, `src/resources/extensions/gsd/prompts/complete-slice.md`, `src/resources/extensions/gsd/prompts/replan-slice.md`, `src/resources/extensions/gsd/prompts/complete-milestone.md`
  - Do: Replace the git command text in each step with a declarative "system auto-commits" message. Keep step numbers intact. Preserve the "do not squash-merge manually" context in complete-slice.md. Do NOT touch worktree-merge.md.
  - Verify: grep returns zero hits; build passes; test passes; worktree-merge.md has no diff
  - Done when: all four verification commands pass

## Files Likely Touched

- `src/resources/extensions/gsd/prompts/execute-task.md`
- `src/resources/extensions/gsd/prompts/complete-slice.md`
- `src/resources/extensions/gsd/prompts/replan-slice.md`
- `src/resources/extensions/gsd/prompts/complete-milestone.md`
