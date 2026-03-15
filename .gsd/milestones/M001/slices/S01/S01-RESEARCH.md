# S01: Manifest Wiring & Prompt Verification — Research

**Date:** 2026-03-12

## Summary

S01's scope is narrow and well-grounded. The critical infrastructure — parser (`parseSecretsManifest`), formatter (`formatSecretsManifest`), types (`SecretsManifest`, `SecretsManifestEntry`), path resolution (`resolveMilestoneFile(base, mid, "SECRETS")`), and prompt instructions (both `plan-milestone.md` and `guided-plan-milestone.md`) — already exist and pass 312 parser tests including a round-trip test. The remaining work is:

1. **Implement `getManifestStatus()`** — a new function that reads the manifest from disk, checks each entry's status against `.env`/`process.env` via existing `checkExistingEnvKeys()`, and returns a categorized status object `{ pending, collected, skipped, existing }`.
2. **Verify prompt compliance** — prove that the plan-milestone prompt's secret forecasting instructions produce output the parser can consume. This is currently untested end-to-end (prompt → manifest file → parser round-trip).
3. **Wire the prompt variable** — `{{secretsOutputPath}}` is already substituted in both auto-mode (`buildPlanMilestonePrompt` in auto.ts:1658-1666) and guided flow (`guided-flow.ts:614-617`). No wiring changes needed.

The main risk is prompt compliance: the LLM might produce manifests with formatting variations the parser doesn't handle. The parser is already forgiving (defaults missing fields to empty/pending), so this risk is low. The round-trip test in `parsers.test.ts` already proves format stability.

## Recommendation

Implement `getManifestStatus()` in a new file or in `files.ts`, backed by `parseSecretsManifest()` + `checkExistingEnvKeys()`. Add contract tests proving:
- `getManifestStatus()` correctly categorizes entries by status and env presence
- A realistic LLM-style manifest (varying whitespace, missing optional fields) round-trips through `parseSecretsManifest() → formatSecretsManifest() → parseSecretsManifest()` with semantic equality

No new libraries needed. No prompt changes needed — the instructions are already in place.

## Don't Hand-Roll

| Problem | Existing Solution | Why Use It |
|---------|------------------|------------|
| Parse secrets manifest | `parseSecretsManifest()` in `files.ts` | Already tested with 312-test suite, handles edge cases (missing fields, invalid status) |
| Format secrets manifest | `formatSecretsManifest()` in `files.ts` | Round-trip tested, produces canonical format |
| Resolve manifest path | `resolveMilestoneFile(base, mid, "SECRETS")` in `paths.ts` | Handles legacy directory names, exact+prefix matching |
| Check existing env keys | `checkExistingEnvKeys()` in `get-secrets-from-user.ts` | Checks both `.env` file and `process.env`, tested |
| Detect destination | `detectDestination()` in `get-secrets-from-user.ts` | File-presence heuristic (vercel.json → Vercel, convex/ → Convex), tested |
| Load file from disk | `loadFile()` in `files.ts` | Returns null on ENOENT instead of throwing |

## Existing Code and Patterns

- `src/resources/extensions/gsd/files.ts` — Contains `parseSecretsManifest()` and `formatSecretsManifest()`. Parser uses `extractAllSections()` at H3 level and `extractBoldField()` for structured fields. `getManifestStatus()` should follow the same pure-function pattern. Also has `loadFile()` for disk I/O and `saveFile()` for atomic writes.
- `src/resources/extensions/gsd/types.ts` (lines 121-136) — `SecretsManifestEntryStatus`, `SecretsManifestEntry`, `SecretsManifest` types already defined. The `getManifestStatus` return type is new and needs to be added here.
- `src/resources/extensions/gsd/paths.ts` — `resolveMilestoneFile(base, mid, "SECRETS")` resolves the manifest path with legacy fallback. Already used in `auto.ts:1658` and `guided-flow.ts:614`.
- `src/resources/extensions/gsd/auto.ts` (lines 1658-1666) — `buildPlanMilestonePrompt()` already passes `secretsOutputPath` to `loadPrompt("plan-milestone", ...)`. No changes needed.
- `src/resources/extensions/gsd/guided-flow.ts` (lines 614-617) — Guided flow already passes `secretsOutputPath` to `loadPrompt("guided-plan-milestone", ...)`. No changes needed.
- `src/resources/extensions/gsd/prompts/plan-milestone.md` (line 69) — Secret forecasting instructions with `{{secretsOutputPath}}` substitution. Well-structured, references the template. No changes needed.
- `src/resources/extensions/gsd/prompts/guided-plan-milestone.md` (line 27) — Same forecasting instructions in guided variant. No changes needed.
- `src/resources/extensions/gsd/templates/secrets-manifest.md` — Template showing expected H3 format with bold fields and numbered guidance. Parser aligns with this format.
- `src/resources/extensions/get-secrets-from-user.ts` — `checkExistingEnvKeys()` (line 105) and `detectDestination()` (line 128) are already exported and tested. `collectOneSecret()` (line 149) has `hint` parameter but NOT `guidance` — guidance rendering is S02 scope.
- `src/resources/extensions/gsd/tests/parsers.test.ts` (lines 1252-1500) — Comprehensive parser tests: full manifest, single-key, empty, missing fields, all status variants, invalid status fallback, and round-trip. All 312 tests pass.
- `src/resources/extensions/gsd/tests/secure-env-collect.test.ts` — Tests for `checkExistingEnvKeys()` and `detectDestination()`. All pass.
- `src/resources/extensions/gsd/state.ts` — `deriveState()` does not reference secrets manifests. State derivation changes are NOT needed for S01 — `getManifestStatus()` is a standalone query function, not part of the dashboard state tree.

## Constraints

- `getManifestStatus()` must be async (calls `loadFile()` and `checkExistingEnvKeys()` which do disk I/O)
- Must import `checkExistingEnvKeys` from `../../get-secrets-from-user.ts` (relative path from gsd/ dir) — this cross-module import already has precedent in the codebase
- Must not modify `state.ts` or `deriveState()` — the manifest status is a separate query, not dashboard state (keeps S01 changes minimal and avoids coupling)
- The function must handle the case where no manifest file exists (return empty/null status)
- Tests must use the existing test pattern: `node:test` with `assert/strict`, temp directories for filesystem isolation, cleanup in `finally` blocks

## Common Pitfalls

- **Confusing "existing" vs "collected"** — A key can be both `status: collected` in the manifest AND present in `.env`. These are separate signals. `existing` means "currently in env right now", `collected` means "was previously collected via the tool". The `getManifestStatus` return must distinguish: `existing` (in env regardless of manifest status), `collected` (manifest says collected, may or may not be in env), `pending` (manifest says pending AND not in env), `skipped` (manifest says skipped).
- **Import path for `checkExistingEnvKeys`** — It's in `src/resources/extensions/get-secrets-from-user.ts`, not in the gsd/ subtree. Import must use the correct relative path from wherever `getManifestStatus` lives.
- **Manifest file might not exist** — `resolveMilestoneFile()` returns `null` when the file doesn't exist on disk. `loadFile()` returns `null` on ENOENT. Both must be handled gracefully.
- **Env file path for `checkExistingEnvKeys`** — Needs a `.env` path. Use `resolve(base, '.env')` consistent with how `secure_env_collect` resolves it.

## Open Risks

- **Prompt compliance remains probabilistic** — The LLM produces the manifest, so formatting could vary. The parser is forgiving (defaults missing fields), but there's no way to guarantee 100% compliance without testing against real LLM output. The round-trip test proves the parser handles the canonical format; edge case tolerance is already tested (missing fields, invalid status). This risk is acceptably low.
- **Cross-module import stability** — `getManifestStatus()` depends on `checkExistingEnvKeys()` from `get-secrets-from-user.ts`. If that module's exports change, this breaks. Low risk — the function is stable and already tested.

## Skills Discovered

| Technology | Skill | Status |
|------------|-------|--------|
| Secrets management | `wshobson/agents@secrets-management` | Available (3.2K installs) — not relevant, generic secrets skill unrelated to pi TUI/GSD internals |

No relevant external skills found. This work is entirely internal to the GSD extension codebase.

## Sources

- Existing parser tests pass (312/312) including round-trip (source: `npx tsx src/resources/extensions/gsd/tests/parsers.test.ts`)
- Existing `checkExistingEnvKeys` tests pass (source: `npx tsx src/resources/extensions/gsd/tests/secure-env-collect.test.ts`)
- Prompt variables already wired in auto.ts:1658-1666 and guided-flow.ts:614-617 (source: code inspection)
- Secret forecasting instructions present in both plan-milestone.md and guided-plan-milestone.md (source: code inspection)
