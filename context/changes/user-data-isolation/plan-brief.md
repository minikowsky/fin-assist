# User data isolation and production migration path (F-01) — Plan Brief

> Full plan: `context/changes/user-data-isolation/plan.md`

## What & Why

Before any slice stores financial data, fin-assist needs a safe, repeatable way to get database migrations into production, and proof that one user can never see another user's data. The PRD guardrail ("no user's financial data is ever visible to another user") can't be fixed after a leak, so the proof has to exist before the first table does.

## Starting Point

There are no migrations, and production Supabase has never received `db push`. Cloudflare Workers Builds deploys every push to `master`, independently of CI. The smoke test signs in one user, and CI already starts a local Supabase.

## Desired End State

Every Builds deploy first applies pending migrations to production, behind a guard that blocks destructive SQL. CI fails if any `public` table breaks the RLS convention or if user B can touch user A's rows. The smoke test runs two users, including against production. Production carries the first migration, which revokes `anon` defaults.

## Key Decisions Made

| Decision | Choice | Why (1 sentence) |
| --- | --- | --- |
| How migrations reach prod | `npm run db:deploy` in the Workers Builds **deploy** command, before `wrangler deploy` | Guarantees migration-before-code ordering, and runs only after lint, check and build pass. |
| Migration/deploy ordering rule | Additive and backward-compatible only (expand/contract) | A Worker rollback doesn't roll back the DB (`infrastructure.md:111`). |
| Destructive-migration safety | Dependency-free guard script in CI and in `db:deploy`, with an `-- allow-destructive: <reason>` override | Auto-apply plus an unprotected `master` needs an enforced gate, not just a documented rule. |
| Isolation check location | pgTAP (`supabase test db`) **and** a two-user smoke harness | RLS is proven where it lives, and end to end against production. |
| What F-01 asserts without domain tables | Fixture table in a rolled-back transaction, plus a catalog guard over all `public` tables | Proves the convention works today and enforces it on every future table. |
| First migration | Revoke `anon` default privileges on future `public` tables and sequences | Real defense in depth that also exercises the production path. |
| Production smoke | Second auto-confirmed production smoke user; B credentials required in existing-account mode | Production isolation can't be skipped silently. |
| CLI version | `supabase` devDependency pinned to 3.2.0 (same as CI) | One CLI version locally, in CI and in Builds. |

## Scope

**In scope:**
- Supabase CLI 3.2.0
- the revoke-anon migration
- `scripts/check-migrations.mjs` (`db:check`)
- pgTAP catalog guard and fixture isolation test (`test:db`)
- two-user `smoke.mjs`
- `scripts/db-deploy.mjs` (`db:deploy`)
- Builds secret and deploy-command change
- CLAUDE.md and `deployment-plan.md` updates

**Out of scope:**
- domain tables
- a GitHub Actions deploy
- Supabase Branching or a preview DB
- a shared SQL test-helper library
- function `EXECUTE` revocation
- sign-in redirect and sign-up changes (S-01)
- branch protection

## Architecture / Approach

push to `master` → Builds build command (`astro sync`, lint, check, build) → deploy command: `db:deploy` (env check → guard → `supabase db push --db-url $SUPABASE_DB_URL --yes` via the IPv4 session pooler) → `wrangler deploy`. A guard or push failure means no deploy.

CI (on PRs) runs `db:check`, then in the smoke job `supabase test db` (catalog guard and fixture isolation), then the two-user smoke test against the workerd preview.

## Phases at a Glance

| Phase | What it delivers | Key risk |
| --- | --- | --- |
| 1. Migration conventions and guard | CLI 3.2.0, revoke-anon migration, `db:check` in CI, CLAUDE.md rules | Guard regex false positives (mitigated by the allowed `DROP POLICY` etc. and the marker) |
| 2. pgTAP RLS tests in CI | Catalog guard plus fixture A/B isolation template, `test:db` | JWT-claim impersonation details in pgTAP |
| 3. Two-user smoke harness | Per-user sessions, B credentials required in production mode, own-email assertion, slice extension point | Merge overlap with S-01's `smoke.mjs` edit |
| 4. Production path via Builds | `db:deploy` wrapper; 👤 smoke user B, Builds secret, dry run, deploy-command switch, production verification | Secret not visible to the deploy command, or leaked into the log; Builds-image connectivity |

**Prerequisites:** Supabase dashboard access (DB password, Users), Cloudflare Builds settings access, password manager. Implementation happens on a feature branch with a PR (preview builds are off).

**Estimated effort:** ~2 sessions across 4 phases. Phases 1–3 are local and CI work; phase 4 is mostly 👤 dashboard steps.

## Open Risks & Assumptions

- Assumes Builds "Variables and secrets" are visible to the deploy command. The fallback is to move `db:deploy` to the end of the build command.
- Assumes the Builds image reaches the session pooler (IPv4) and that the `supabase` npm postinstall can download its binary there.
- Auto-applied migrations replace the earlier "human-only data-dropping migrations" rule. The guard and marker are the new control, and a marker in a reviewed PR is the only path for a destructive change.
- If the DB password ever shows up in a Builds log, rotate it immediately.

## Success Criteria (Summary)

- A push to `master` applies pending migrations to production before the Worker deploys, and refuses destructive SQL without an explicit marker.
- CI proves, on every PR, that a convention-built table isolates user A from user B, and rejects any `public` table that breaks the RLS convention.
- The production smoke test signs in two real users and confirms that each sees only their own identity, ready for S-03+ to add data assertions.
