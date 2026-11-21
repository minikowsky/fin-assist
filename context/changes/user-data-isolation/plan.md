# User data isolation and production migration path (F-01) Implementation Plan

## Overview

Roadmap foundation F-01 (`context/foundation/roadmap.md`, change `user-data-isolation`). Before the first slice stores financial data (S-03, S-04, S-06), put three things in place:

1. a repeatable, automatic path for Supabase migrations to reach production, guarded against destructive changes;
2. the "every user-owned table ships with per-operation RLS" convention, enforced in CI;
3. a two-user isolation check, at the database level (pgTAP) and end-to-end (smoke test, also against production).

No domain tables are created. Each slice adds its own tables and extends the checks built here.

## Current State Analysis

- **No migrations exist.** There is no `supabase/migrations/` directory and no `supabase/seed.sql`. Production Supabase (`ckvcyftxbatipvjhrome`, eu-north-1, Free) is linked locally but has never received `db push` (`context/changes/deployment/deployment-plan.md:449`, open follow-up).
- **Deploys are decoupled from CI.** Cloudflare Workers Builds deploys every push to `master` (build `npx astro sync && npm run lint && npx astro check && npm run build`, deploy `npx wrangler deploy`), independently of GitHub Actions. `master` is unprotected (CLAUDE.md "CI"; `deployment-plan.md:440`).
- **Fix-forward policy already stated.** Migrations must be backward-compatible with the previous Worker version, because a Worker rollback doesn't roll back the DB (`context/foundation/infrastructure.md:111`). Until now, data-dropping production migrations were human-only (`infrastructure.md:115`). This plan replaces that with an enforced guard plus an explicit override marker.
- **Smoke test is single-user.** `scripts/smoke.mjs:6-9` has one cookie jar. It signs up a fresh user (local/CI, where `enable_confirmations = false`), or signs in with `SMOKE_EMAIL`/`SMOKE_PASSWORD` (production). `request()` returns only status and location (`scripts/smoke.mjs:25-38`).
- **CI already runs a local Supabase.** `.github/workflows/ci.yml:116-123` uses `supabase/setup-cli@v1` pinned to `3.2.0` and runs `supabase start` (which applies migrations), then preview + smoke. No test runner is installed in the repo. pgTAP via `supabase test db` is available in the local Supabase stack.
- **CLI version split.** The npm devDependency `supabase` resolves to `2.117.0` (`package-lock.json:9395-9397`), while CI pins `3.2.0`. `supabase@3.2.0` exists on npm. `db push` supports `--db-url`, `--dry-run` and the global `--yes`.
- **The dashboard shows the signed-in user's email** (`src/pages/dashboard.astro:13`). That gives the smoke test something user-specific to assert today.
- **No `context/foundation/lessons.md`** and no `docs/reference/contract-surfaces.md` exist yet.

## Desired End State

- Every push to `master` that reaches the Workers Builds deploy step first runs `npm run db:deploy`. It fails if `SUPABASE_DB_URL` is missing, runs the destructive-migration guard, then applies pending migrations to production. Only then does `wrangler deploy` run. A failed guard or push means no deploy.
- Production Supabase has a migration history containing the first migration, which revokes `anon`'s default privileges on future `public` tables and sequences.
- CI fails if:
  - a migration file is misnamed or contains a destructive statement without `-- allow-destructive: <reason>`;
  - any `public` table lacks RLS, has no policy, uses a `FOR ALL` policy, or has a policy for `anon`/`public`;
  - the fixture isolation test shows user B can read or modify user A's rows.
- `npm run smoke` signs in two users. Against production it requires both A and B credentials and asserts each session sees only its own identity. It has a marked section where slices append cross-user data assertions.
- CLAUDE.md documents the migration conventions (additive-only, RLS-per-operation, override marker, test template) and the new deploy path.

Verify by:
- the CI run on the PR (lint, `db:check`, `test:db`, two-user smoke all green);
- the first Workers Builds log after the deploy-command change (migration applied, no secret printed);
- `supabase migration list` against production;
- the production two-user smoke run.

### Key Discoveries:

- `scripts/smoke.mjs:43-51`: signup step is skipped in existing-account mode. The two-user refactor must keep the plain-signup CI mode unchanged in behavior.
- `.github/workflows/ci.yml:122`: CI starts Supabase with most services excluded. `supabase test db` only needs the DB, so the step fits in the existing smoke job after `supabase start`.
- `supabase/config.toml:5`: `project_id = "10x-astro-starter"`. Harmless for `db push --db-url`; not renamed here.
- `deployment-plan.md:287`: never `supabase config push` to production. `db push` doesn't touch auth config, so this rule still holds.
- `deployment-plan.md:344`: `SUPABASE_KEY` must stay the anon/publishable key. The new `SUPABASE_DB_URL` is a **Builds-only** secret, never a Worker runtime secret.

## What We're NOT Doing

- No domain tables (sets, records, categories, rules). Each slice adds its own.
- No GitHub Actions deploy or migration job; Workers Builds stays the only production path.
- No Supabase Branching / GitHub integration and no preview database.
- No shared SQL test-helper library (e.g. basejump `supabase_test_helpers`). The fixture test is the copyable template.
- No revocation of default function `EXECUTE` privileges (granted to `PUBLIC`), and no change to existing Supabase-owned grants.
- No changes to the sign-in redirect target or sign-up. That is S-01, which also edits `scripts/smoke.mjs` (expect a small merge).
- No branch protection on `master`. It stays an open follow-up in `deployment-plan.md`.
- No renaming of `project_id` in `supabase/config.toml`.

## Implementation Approach

Build the local, reversible pieces first (conventions, guard, tests, smoke), all verified in CI on a feature branch (`user-data-isolation`, PR to `master`). Preview builds are off, so branch pushes don't deploy. The production rollout goes last, in an order that never breaks Builds:

1. Merge the code; the deploy command is still the old one, so no migration runs.
2. Add the Builds secret and do a local dry run.
3. Switch the deploy command.
4. Retry the build, which applies the migration.

Migrations run in the **deploy** command, after the build command's quality gates pass, so a lint or type failure never migrates production.

## Critical Implementation Details

- **Timing & lifecycle:** The Workers Builds deploy command must not reference `npm run db:deploy` until the commit that adds that script is on `master`. Otherwise every build fails. If `SUPABASE_DB_URL` turns out not to be visible to the deploy command (verify in the first Builds log), move `npm run db:deploy` to the end of the build command instead. It still runs only after lint, check and build pass.
- **Connection string:** Use the **Session pooler** URL (IPv4, port 5432, user `postgres.ckvcyftxbatipvjhrome`) from Dashboard → Connect, with the password percent-encoded. The direct connection is IPv6-only, and the Builds image's IPv6 support is unverified. The wrapper must never echo the URL. Check the first Builds log for a leaked password, and rotate the DB password if one appears.
- **pgTAP impersonation:** `auth.uid()` reads the `sub` claim from `request.jwt.claims`. Each step that acts as a user must `set local role authenticated` and `set_config('request.jwt.claims', '{"sub":"<uuid>","role":"authenticated"}', true)`, after inserting the two users into `auth.users`. Everything stays inside `begin … rollback`.

## Phase 1: Migration conventions and destructive-migration guard

### Overview

Align the CLI version, add the first real migration, add the dependency-free guard script and run it in CI, and document the conventions.

### Changes Required:

#### 1. Supabase CLI version

**File**: `package.json`, `package-lock.json`

**Intent**: Bump the `supabase` devDependency to `3.2.0` so local, CI (`setup-cli` 3.2.0) and Workers Builds (`npx supabase`) all run the same CLI.

**Contract**: `devDependencies.supabase` = `3.2.0` (exact pin, like `husky`); lockfile regenerated with `npm install`.

#### 2. First migration: revoke anon defaults

**File**: `supabase/migrations/<YYYYMMDDHHmmss>_revoke_anon_default_privileges.sql`

**Intent**: Defense in depth. The app only reads data as `authenticated`, so future `public` tables and sequences created by migrations (owned by `postgres`) should grant nothing to `anon`. RLS remains the per-row gate. This is also the first migration that proves the production path.

**Contract**: The header comment states the purpose. Default privileges only; no objects created. Any future public-read feature must grant to `anon` explicitly.

```sql
alter default privileges for role postgres in schema public revoke all on tables from anon;
alter default privileges for role postgres in schema public revoke all on sequences from anon;
```

#### 3. Destructive-migration guard

**File**: `scripts/check-migrations.mjs` (new), `package.json`

**Intent**: Enforce the additive-only rule at every point where migrations are verified or applied, because Builds applies them to production automatically. It is dependency-free (Node built-ins only), like `smoke.mjs`.

**Contract**:
- `npm run db:check` → `node scripts/check-migrations.mjs`. It scans every `supabase/migrations/*.sql`, exits `0` when clean and `1` with `file:line` and the offending statement otherwise. It also passes when the directory has no migrations.
- Filenames must match `^\d{14}_[a-z0-9_]+\.sql$`.
- SQL comments are stripped before matching, but the marker is read from the raw file.
- These patterns are blocked (case-insensitive):
  - `DROP TABLE|SCHEMA|COLUMN|VIEW|MATERIALIZED VIEW|TYPE`
  - `TRUNCATE`
  - `RENAME` (any `ALTER … RENAME`)
  - `ALTER COLUMN … TYPE` / `ALTER COLUMN … SET DATA TYPE`
  - `DELETE FROM`
- `DROP POLICY|TRIGGER|FUNCTION|INDEX|CONSTRAINT` stay allowed, because they don't lose data and replacing policies needs them.
- A file containing `-- allow-destructive: <non-empty reason>` is exempt, and the script prints the reason as a notice.

#### 4. CI

**File**: `.github/workflows/ci.yml`

**Intent**: Run the guard on every push and PR.

**Contract**: The `ci` job gets a `npm run db:check` step after `npm ci`.

#### 5. Conventions

**File**: `CLAUDE.md`

**Intent**: Make the rules discoverable to agents and humans.

**Contract**: Extend the "Supabase migrations" bullet in **Key conventions**:
- migrations are applied to production automatically on push to `master`;
- migrations must be additive and backward-compatible with the running Worker (expand/contract for destructive changes);
- the `-- allow-destructive: <reason>` override, and when it's acceptable;
- every user-owned table gets RLS plus separate `select`/`insert`/`update`/`delete` policies `to authenticated`, keyed on `(select auth.uid()) = user_id`, with no `for all` and no `anon` policies;
- every new table ships with a pgTAP isolation test copied from the Phase 2 template.

Add `npm run db:check` to **Commands**.

### Success Criteria:

#### Automated Verification:

- `npx supabase --version` prints `3.2.0` after `npm ci`
- `npx supabase db reset` applies the migration cleanly on local Supabase
- `npm run db:check` passes on the repo's migrations
- `npm run db:check` fails on a temporary migration containing `drop table x;`, passes once that file carries `-- allow-destructive: test`, and the temp file is removed afterwards
- `npm run lint && npx astro check && npm run build` pass

#### Manual Verification:

- CLAUDE.md migration conventions read clearly and match the guard's actual rules

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 2: pgTAP RLS tests in CI

### Overview

Add a DB-level catalog guard that enforces the RLS convention on every `public` table, and a fixture-table isolation test that proves the convention isolates two users. The fixture test is the template slices copy.

### Changes Required:

#### 1. RLS catalog guard

**File**: `supabase/tests/database/rls_catalog_guard.test.sql` (new)

**Intent**: Fail CI the moment any slice adds a `public` table that breaks the convention. It passes vacuously while there are no tables.

**Contract**: pgTAP inside `begin … rollback` (`create extension if not exists pgtap with schema extensions` inside the transaction). It asserts, over `pg_class` (relkind `r`/`p`) in schema `public` joined to `pg_policies`, that the result sets of the following are empty:
- tables with `relrowsecurity = false`;
- tables with zero policies;
- policies with `cmd = 'ALL'`;
- policies whose `roles` include `anon` or `public`.

Failure messages name the offending table or policy.

#### 2. Fixture isolation test (template)

**File**: `supabase/tests/database/rls_isolation_fixture.test.sql` (new)

**Intent**: Prove, today, that a table built by the convention isolates user A from user B, and that the Phase 1 migration leaves `anon` with no table privileges. A header comment explains how slices copy it for real tables.

**Contract**:
- Everything runs inside one `begin … rollback`.
- Setup:
  - create `public.rls_fixture` (`id`, `user_id uuid not null default auth.uid() references auth.users`, `note text`);
  - enable RLS;
  - add four per-operation policies `to authenticated` on `(select auth.uid()) = user_id` (with `with check` on insert/update);
  - insert users A and B into `auth.users`.
- Assertions:
  - `anon` has no `select` privilege on `rls_fixture`;
  - `authenticated` does;
  - as A, insert a row; as A, it is visible;
  - as B:
    - `select` returns 0 rows;
    - `update` and `delete` of A's row affect 0 rows;
    - an `insert` with `user_id = A` throws an RLS violation;
  - as A, the row is still unchanged.

#### 3. Script and CI

**File**: `package.json`, `.github/workflows/ci.yml`

**Intent**: One command locally, the same step in CI.

**Contract**: `npm run test:db` → `supabase test db`. The `smoke` job gets a `supabase test db` step right after "Start local Supabase", before build.

### Success Criteria:

#### Automated Verification:

- `npm run test:db` passes locally, running both test files
- Catalog guard fails when a temporary migration adds a `public` table without RLS, and passes again once it is removed (`npx supabase db reset` between runs)
- `npm run db:check` and `npm run lint` still pass
- The PR's CI run is green, including the new `supabase test db` step (`gh pr checks`)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 3: Two-user smoke harness

### Overview

Refactor `scripts/smoke.mjs` so it runs two independent user sessions. Add a first cross-user assertion and an extension point for slices.

### Changes Required:

#### 1. Per-user sessions

**File**: `scripts/smoke.mjs`

**Intent**: Each user gets its own cookie jar, so slices can assert that "B never sees A's data" over HTTP, including against production. It stays dependency-free.

**Contract**:
- A `createSession(email, password)` (or equivalent) owns a cookie jar and a `request(path, opts)` that also returns the response `body` text.
- Users `A` and `B`:
  - In signup mode (no credentials set), both are fresh `smoke-<ts>-a|b@example.com` accounts, with the existing signup step run for each.
  - In existing-account mode, `SMOKE_EMAIL`/`SMOKE_PASSWORD` and `SMOKE_EMAIL_B`/`SMOKE_PASSWORD_B` are **all** required. If only some are set, it exits `1` before any request, with a message naming the missing variables, so production isolation can't be skipped silently.
- The existing steps keep their expectations, including sign-in → `/`, which S-01 will change.
- New steps, with both users signed in at the same time:
  - A's `/dashboard` body contains A's email and not B's;
  - B's `/dashboard` body contains B's email and not A's;
  - A's `/dashboard` is re-checked after B signed in.
- A clearly marked "Cross-user isolation steps" block, with a comment, is where slices append data assertions.
- Both users are signed out at the end.
- The header comment documents the new env vars.

#### 2. Docs

**File**: `CLAUDE.md`

**Intent**: Keep the smoke command description accurate.

**Contract**: The `npm run smoke` line mentions the two-user mode and the `SMOKE_EMAIL_B`/`SMOKE_PASSWORD_B` requirement for existing-account runs.

### Success Criteria:

#### Automated Verification:

- Against a local preview (`npm run build`, `npm run preview`, local Supabase), `npm run smoke` passes every step, including the two-user dashboard steps
- `SMOKE_EMAIL=x SMOKE_PASSWORD=y npm run smoke` (B unset) exits non-zero before any request and names the missing variables
- Existing-account mode with two locally created users passes
- `npm run lint` passes
- The PR's CI smoke job is green

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 4: Production migration path through Workers Builds

### Overview

Add the `db:deploy` wrapper, merge, then wire it into Workers Builds and roll out the first migration to production. Steps marked 👤 are production mutations or secrets, so the human runs or approves them (per `deployment-plan.md` conventions).

### Changes Required:

#### 1. Deploy wrapper

**File**: `scripts/db-deploy.mjs` (new), `package.json`

**Intent**: A single, safe entry point for Builds and for a manual escape hatch.

**Contract**:
- `npm run db:deploy` → `node scripts/db-deploy.mjs`. In order:
  1. exit `1` with a clear message if `SUPABASE_DB_URL` is unset or empty;
  2. run the `check-migrations` logic (exit `1` on failure);
  3. run `supabase db push --db-url "$SUPABASE_DB_URL" --yes` via `child_process`, with inherited stdio for the CLI's own output.
- It never prints the URL.
- It propagates the CLI's exit code.
- Optional `--dry-run` passthrough for a 👤 pre-check.

#### 2. Production rollout (👤 steps, in this order)

**File**: n/a (Cloudflare and Supabase dashboards)

**Intent**: Enable automatic migrations without ever breaking a Builds run.

**Contract**:
1. 👤 Merge the PR to `master`. Builds deploys with the **old** deploy command, so no migration runs.
2. 👤 Supabase dashboard → Authentication → Users → add the second smoke user with **Auto Confirm**, and store `SMOKE_EMAIL_B`/`SMOKE_PASSWORD_B` in the password manager.
3. 👤 Copy the Session pooler URL (percent-encoded password) and add it in Cloudflare → `fin-assist` → Settings → Builds → Variables and secrets as **encrypted** `SUPABASE_DB_URL`. Never as a Worker runtime secret.
4. 👤 Locally, run `SUPABASE_DB_URL=… npm run db:deploy -- --dry-run`. It should list exactly the one migration.
5. 👤 Change the Builds deploy command to `npm run db:deploy && npx wrangler deploy`.
6. 👤 Build History → Retry build on `master`.

#### 3. Docs and trackers

**File**: `CLAUDE.md`, `context/changes/deployment/deployment-plan.md`

**Intent**: Record the new deploy path so nobody bypasses or duplicates it.

**Contract**:
- CLAUDE.md:
  - The **CI** section's Builds deploy command becomes `npm run db:deploy && npx wrangler deploy`. It notes the `SUPABASE_DB_URL` Builds secret and that migrations apply before each deploy.
  - **Commands** gains `npm run db:deploy` and `npm run test:db`.
- `deployment-plan.md`:
  - tick the "First table: `supabase db push`" follow-up with the outcome (date, version ID);
  - add the deploy command and the `SUPABASE_DB_URL` Builds secret to the **Outcome** table;
  - update the **Rollback target**.

### Success Criteria:

#### Automated Verification:

- `npm run db:deploy` with `SUPABASE_DB_URL` unset exits non-zero with a clear message
- `SUPABASE_DB_URL=postgresql://postgres:postgres@127.0.0.1:54322/postgres npm run db:deploy` succeeds against local Supabase, and a second run reports nothing to push
- `npm run db:check`, `npm run lint`, `npx astro check` and `npm run build` pass

#### Manual Verification:

- Second production smoke user exists and its credentials are in the password manager
- `SUPABASE_DB_URL` is set as an encrypted Builds variable, not a Worker secret (`npx wrangler secret list` still shows only `SUPABASE_URL`, `SUPABASE_KEY`)
- Production dry run lists exactly the revoke-anon migration
- The retried Builds run succeeds, its log shows the migration applied before `wrangler deploy`, and no password appears in the log
- `npx supabase migration list --db-url "$SUPABASE_DB_URL"` shows the migration applied remotely
- `BASE_URL=https://fin-assist.minik.workers.dev SMOKE_EMAIL=… SMOKE_PASSWORD=… SMOKE_EMAIL_B=… SMOKE_PASSWORD_B=… npm run smoke` passes all steps
- CLAUDE.md and `deployment-plan.md` reflect the new deploy command, the secret and the new rollback target

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Testing Strategy

### Unit Tests:

- None (no unit test runner in the repo). The guard script is checked by the pass, fail and override cases in Phase 1.

### Integration Tests:

- pgTAP via `supabase test db`:
  - the catalog guard (convention on every `public` table);
  - the fixture isolation test (A vs B for select, update, delete and insert-as-other; anon privileges).
- Smoke via `npm run smoke`: two concurrent sessions, each seeing only its own identity, in CI (signup mode) and in production (existing accounts).

### Manual Testing Steps:

1. Run the production dry run before switching the deploy command.
2. Read the first post-switch Builds log for migration output and secret leakage.
3. Run the production two-user smoke test.

## Performance Considerations

- `db push` adds a few seconds to each Builds deploy (a no-op when nothing is pending).
- None at runtime.

## Migration Notes

- The first `db push` creates `supabase_migrations.schema_migrations` in production and applies one privilege-only migration. No data is touched.
- Rollback:
  - a Worker rollback (`wrangler rollback`) is unaffected;
  - to undo the migration, fix forward with a new migration that re-grants;
  - to stop automatic migrations, revert the Builds deploy command to `npx wrangler deploy`.
- If the DB password ever leaks into a log, rotate it in Dashboard → Database → Settings and update `SUPABASE_DB_URL`.

## References

- Roadmap item: `context/foundation/roadmap.md` (F-01)
- PRD guardrails and visibility: `context/foundation/prd.md:49-52`, `context/foundation/prd.md:131-140`
- Deployment tracker and follow-ups: `context/changes/deployment/deployment-plan.md:443-452`
- Fix-forward rule: `context/foundation/infrastructure.md:111-115`
- Smoke test: `scripts/smoke.mjs`
- CI: `.github/workflows/ci.yml`

## Progress

> Convention: `- [ ]` pending, `- [x]` done. Append ` — <commit sha>` when a step lands. Do not rename step titles. See `references/progress-format.md`.

### Phase 1: Migration conventions and destructive-migration guard

#### Automated

- [ ] 1.1 `npx supabase --version` prints `3.2.0` after `npm ci`
- [ ] 1.2 `npx supabase db reset` applies the migration cleanly on local Supabase
- [ ] 1.3 `npm run db:check` passes on the repo's migrations
- [ ] 1.4 `npm run db:check` fails on a temporary destructive migration, passes with the override marker, temp file removed
- [ ] 1.5 `npm run lint && npx astro check && npm run build` pass

#### Manual

- [ ] 1.6 CLAUDE.md migration conventions read clearly and match the guard's actual rules

### Phase 2: pgTAP RLS tests in CI

#### Automated

- [ ] 2.1 `npm run test:db` passes locally, running both test files
- [ ] 2.2 Catalog guard fails on a temporary table without RLS and passes again once removed
- [ ] 2.3 `npm run db:check` and `npm run lint` still pass
- [ ] 2.4 The PR's CI run is green, including the new `supabase test db` step

### Phase 3: Two-user smoke harness

#### Automated

- [ ] 3.1 Local `npm run smoke` passes every step, including the two-user dashboard steps
- [ ] 3.2 Existing-account mode with B unset exits non-zero before any request and names the missing variables
- [ ] 3.3 Existing-account mode with two locally created users passes
- [ ] 3.4 `npm run lint` passes
- [ ] 3.5 The PR's CI smoke job is green

### Phase 4: Production migration path through Workers Builds

#### Automated

- [ ] 4.1 `npm run db:deploy` with `SUPABASE_DB_URL` unset exits non-zero with a clear message
- [ ] 4.2 `npm run db:deploy` against local Supabase succeeds and a second run reports nothing to push
- [ ] 4.3 `npm run db:check`, `npm run lint`, `npx astro check` and `npm run build` pass

#### Manual

- [ ] 4.4 Second production smoke user exists and its credentials are in the password manager
- [ ] 4.5 `SUPABASE_DB_URL` is an encrypted Builds variable, not a Worker secret
- [ ] 4.6 Production dry run lists exactly the revoke-anon migration
- [ ] 4.7 Retried Builds run succeeds, log shows the migration applied before deploy, no password in the log
- [ ] 4.8 `supabase migration list` shows the migration applied remotely
- [ ] 4.9 Production two-user smoke passes all steps
- [ ] 4.10 CLAUDE.md and `deployment-plan.md` reflect the new deploy command, secret and rollback target
