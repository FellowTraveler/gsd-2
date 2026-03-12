---
id: T01
parent: S04
milestone: M001
provides:
  - Raw git commands removed from all four LLM-facing prompt files
key_files:
  - src/resources/extensions/gsd/prompts/execute-task.md
  - src/resources/extensions/gsd/prompts/complete-slice.md
  - src/resources/extensions/gsd/prompts/replan-slice.md
  - src/resources/extensions/gsd/prompts/complete-milestone.md
  - src/resources/extensions/gsd/tests/replan-slice.test.ts
key_decisions:
  - Removed unused {{blockerTaskId}} template variable from replan-slice.md (was only used in the git commit message); kept the variable in auto.ts since it's still used in inlined context headings
patterns_established:
  - Prompt steps that previously contained git commands now use declarative "system auto-commits" messages
observability_surfaces:
  - none
duration: 8 minutes
verification_result: passed
completed_at: 2026-03-12
blocker_discovered: false
---

# T01: Replace raw git commands in all four prompt files

**Replaced git add/commit/update-index/hash-object instructions in execute-task.md, complete-slice.md, replan-slice.md, and complete-milestone.md with declarative auto-commit messages.**

## What Happened

Replaced the raw git command text in step 16 (execute-task.md), step 10 (complete-slice.md), step 6 (replan-slice.md), and step 9 (complete-milestone.md) with short declarative messages telling the LLM not to commit manually. The execute-task.md replacement also removed the `git update-index`/`git hash-object` workaround text. The complete-slice.md replacement preserves the squash-merge context.

Discovered that the replan-slice test asserted `{{blockerTaskId}}` rendered in the prompt — that variable was only present in the removed git commit message line. Removed the stale `blockerTaskId` assertion and the corresponding template var from that test case.

## Verification

- `grep -rn 'git add\|git commit\|git update-index\|git hash-object' src/resources/extensions/gsd/prompts/ --include="*.md" | grep -v worktree-merge.md` → zero results ✓
- `git diff src/resources/extensions/gsd/prompts/worktree-merge.md` → no changes ✓
- `npm run build` → passes ✓
- `npm run test` → 114 pass / 4 fail (same 4 pre-existing failures as before changes; replan-slice test now passes) ✓

## Diagnostics

None — this task modifies static prompt templates with no runtime behavior.

## Deviations

Updated `src/resources/extensions/gsd/tests/replan-slice.test.ts` to remove the `blockerTaskId` assertion and template var from the prompt-variable-substitution test. This was not in the original plan but was necessary because removing `{{blockerTaskId}}` from the template broke the test.

## Known Issues

None.

## Files Created/Modified

- `src/resources/extensions/gsd/prompts/execute-task.md` — Step 16 replaced with auto-commit message
- `src/resources/extensions/gsd/prompts/complete-slice.md` — Step 10 replaced with auto-commit + merge message
- `src/resources/extensions/gsd/prompts/replan-slice.md` — Step 6 replaced with auto-commit message
- `src/resources/extensions/gsd/prompts/complete-milestone.md` — Step 9 replaced with auto-commit message
- `src/resources/extensions/gsd/tests/replan-slice.test.ts` — Removed stale blockerTaskId assertion
