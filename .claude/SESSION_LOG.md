# Session log: Quincy login fix + CI unblock

Written 2026-09-09 by a Claude Code session, for a future session to pick up context without re-deriving it.

## Task 1: Quincy couldn't log in — FIXED, PR open

**Root cause:** Quincy's `users` row had a tenant reference (`tenantId`/`defaultTenantId`) but no matching `user_tenants` membership row, so `AuthSigninService.resolveSigninTenant` found no active workspace and signin was rejected with "No active workspace available." Same class of bug a prior one-off migration (`packages/server/src/database/system/migrations/20260320000002_backfill_user_tenants_from_users.js`) fixed for existing accounts, but it recurs for any account whose `user_tenants` row was never created (e.g. the signup event that creates it never fired).

**Fix:** `packages/server/src/modules/Auth/commands/AuthSignin.service.ts` — `tryGetActiveTenantForUser` now self-heals: if the user's own tenant reference points at a real, active tenant but the membership row is missing, it creates it on the fly instead of permanently locking the account out.

- Branch: `claude/quincy-login-fix-rb4o2u`
- PR: https://github.com/naoswilbrink/bigcapital/pull/1 (open, ready for review, not yet merged)
- Status as of last check: mergeable, no conflicts. CI (`ESLint Check`, `TypeScript Type Check`) still pending/red — blocked on PR #2 below landing on `develop` first, then this branch needs a rebase.

## Task 2: CI was broken for reasons unrelated to Task 1 — mostly FIXED, PR open

While chasing green CI for PR #1, found CI itself was broken on `develop`:

1. **ESLint Check** failing in `shared/email-components` (5 errors) and `shared/pdf-templates` (11 errors) — unused vars/imports, `@typescript-eslint/no-explicit-any`. One was a real bug: `shared/pdf-templates/src/lib/layout/Stack.tsx` hardcoded `justifyContent="justify"` (invalid CSS) instead of using the `justify` prop. All fixed.

2. **TypeScript Type Check** failing because `.github/workflows/typecheck.yml` never built `@bigcapital/sdk-ts` before `tsc --noEmit`, so every webapp file importing it failed with `Cannot find module`. Added `--scope "@bigcapital/sdk-ts"` to the build step.

3. Fixing #2 surfaced **53 further pre-existing typecheck errors** across `packages/webapp` (previously masked by the sdk-ts failures short-circuiting the run). All fixed — mostly SDK-generated-type vs. hand-rolled-shape mismatches, stale `@ts-expect-error` directives, and Ramda `compose`/`when` pipelines whose types can't express a runtime-branching column-builder pattern (worked around with a documented, locally-scoped `any`-typed alias — see comments in the affected `dynamicColumns.tsx` files). Two more real bugs fixed along the way:
   - `BalanceSheetPdfDialogContent`'s preview/download buttons were permanently disabled (`disabled={!isLoaded}` against a hook that never returned `isLoaded`).
   - The `Stack.tsx` bug above.

- Branch: `claude/fix-ci-lint-typecheck`
- PR: https://github.com/naoswilbrink/bigcapital/pull/2 (open, ready for review, not yet merged)
- Verified locally: `pnpm run build --scope "@bigcapital/utils" --scope "@bigcapital/email-components" --scope "@bigcapital/pdf-templates" --scope "@bigcapital/sdk-ts"` succeeds; `pnpm run typecheck` passes clean across all 3 packages (server, webapp, sdk-ts); `eslint .` in the two shared packages reports 0 errors.

### NOT fixed — explicit, user-approved scope cut

Fixing the two shared packages let CI's lint step (`--nx-bail`) proceed further than it ever had before and reach `packages/server` and `packages/webapp`, which had **never actually been linted in CI**. That revealed:

- `packages/server`: **881 pre-existing lint errors**
- `packages/webapp`: **369 pre-existing lint errors**

The user was asked and explicitly chose **"stop here, merge what's fixed"** rather than grind through 1,250 more errors. So: `ESLint Check` will keep showing red on both PR #1 and PR #2 for this reason, and that's expected/accepted — do not attempt to fix it without the user asking again. If asked to pick this up, it's a large, separate effort (repo-wide lint cleanup across both packages, nothing to do with login or the sdk-ts build) — get explicit sizing/scoping agreement before starting.

## What a follow-up session should do

1. Check current state of PR #1 and PR #2 (CI status, reviews, merge state) — don't assume the above snapshot is still accurate.
2. If PR #2 is green/approved: merge it to `develop`, then rebase/merge `develop` into PR #1's branch so its typecheck goes green too (its lint check will likely still show red — expected, see above).
3. Both PRs were being watched via `subscribe_pr_activity` with periodic check-ins scheduled via `send_later`. If this is a fresh session, re-subscribe if you want automatic wake-ups on new activity.
4. "Push live" / actual deploy: merging to `develop` auto-builds and pushes `:develop` Docker images (see `.github/workflows/build-deploy-develop-container.yaml`). A real production (`:latest`) release needs a GitHub Release or manual `workflow_dispatch` on `build-deploy-container.yml`, which needs Docker Hub/GHCR secrets not verified as configured on this fork — flag that to the user before attempting it.
5. Nobody has confirmed Quincy can actually log in yet — that requires either the user/Quincy trying it after deploy, or DB access to confirm the `user_tenants` row now exists, neither of which this session had.
