<!-- PLAN-REVIEW-REPORT -->
# Plan Review: User data isolation and production migration path (F-01)

- **Plan**: context/changes/user-data-isolation/plan.md
- **Mode**: Deep
- **Date**: 2026-11-23
- **Verdict**: REVISE (before triage) → SOUND (after fixes)
- **Findings**: 1 critical, 4 warnings, 3 observations

## Verdicts

| Dimension | Verdict |
|-----------|---------|
| End-State Alignment | WARNING |
| Lean Execution | PASS |
| Architectural Fitness | PASS |
| Blind Spots | FAIL |
| Plan Completeness | WARNING |

## Grounding

8/8 paths ✓, 5/6 symbols ✓ (`db push --db-url` deprecated in CLI 3.2.0), brief↔plan ✓, 7 stale line refs ✗. Progress↔Phase: 4/4 phases ✓, 25/25 steps ✓.

## Findings

### F1 — Grant model doesn't match hosted Supabase

- **Severity**: ❌ CRITICAL
- **Impact**: 🔎 MEDIUM — real tradeoff; pause to reason through it
- **Dimension**: Blind Spots
- **Location**: Phase 1 §2 and §5, Phase 2 §2
- **Detail**: Since 2026-10-30, hosted Supabase no longer grants anon, authenticated or service_role access to new `public` tables (changelog #45329). CLI 3.2.0 keeps auto-granting locally unless `[api] auto_expose_new_tables = false` is set, and `config.toml` doesn't set it. The fixture's "authenticated has select" check passes in CI but would be false in production, so a slice that copies the template gets "permission denied", and only in production.
- **Fix**: Set `auto_expose_new_tables = false`, require explicit `grant … to authenticated` in the convention and the template, and add a 👤 read-only `pg_default_acl` check on production.
  - Strength: Works the same locally, in CI and in production, whatever the platform default.
  - Tradeoff: One more line in each table migration; the local stack needs a restart.
  - Confidence: MED — the CLI 3.2.0 config template confirms the setting; production ACL not queried.
  - Blind spot: Whether older projects keep the old default. Explicit grants work either way.
- **Decision**: FIXED

### F2 — RLS checks don't gate the production migration

- **Severity**: ⚠️ WARNING
- **Impact**: 🔬 HIGH — architectural stakes; think carefully before deciding
- **Dimension**: End-State Alignment
- **Location**: Phase 4 / What We're NOT Doing
- **Detail**: The catalog guard and the fixture test run only in GitHub Actions, but Builds migrates and deploys every push to `master` regardless of CI. The brief's "marker in a reviewed PR is the only path" wasn't enforced.
- **Fix A ⭐ Recommended**: Branch ruleset on `master`: PR, with `ci` and `smoke` required, no bypass.
  - Strength: Closes both the RLS gap and the marker gap with one setting.
  - Tradeoff: Every change needs a PR and green CI.
  - Confidence: HIGH — rulesets are free for public repos; job names exist in ci.yml.
  - Blind spot: Builds still doesn't read CI status itself.
- **Fix B**: Static RLS check in `check-migrations.mjs`, also run in `db:deploy`.
  - Strength: Gates even a direct push.
  - Tradeoff: Regex heuristic; doesn't close the marker gap.
  - Confidence: MED.
  - Blind spot: False positives when a table is set up across several files.
- **Decision**: FIXED (Fix A)

### F3 — Destructive guard misses some data-losing statements

- **Severity**: ⚠️ WARNING
- **Impact**: 🏃 LOW — quick decision; fix is obvious and narrowly scoped
- **Dimension**: Blind Spots
- **Location**: Phase 1 §3
- **Detail**: CASCADE on allowed drops, plus DROP EXTENSION, SEQUENCE, DOMAIN, OWNED and ROLE, weren't covered.
- **Fix**: A DROP allowlist (POLICY, TRIGGER, FUNCTION, INDEX, CONSTRAINT, DEFAULT, NOT NULL), CASCADE blocked everywhere, UPDATE explicitly allowed.
- **Decision**: FIXED

### F4 — Migration history drift blocks every deploy, with no recovery steps

- **Severity**: ⚠️ WARNING
- **Impact**: 🏃 LOW — quick decision; fix is obvious and narrowly scoped
- **Dimension**: Blind Spots
- **Location**: Phase 4 §1, Phase 1 §5
- **Detail**: `db push` refuses out-of-order timestamps, and a production migration that's missing from the repo; either one fails every Builds run on `master`.
- **Fix**: CLAUDE.md conventions cover timestamp renames, running manual `db:deploy` only from `master`, and recovery (rename or `migration repair`, never `--include-all`).
- **Decision**: FIXED

### F5 — Catalog guard ignores views

- **Severity**: ⚠️ WARNING
- **Impact**: 🏃 LOW — quick decision; fix is obvious and narrowly scoped
- **Dimension**: Blind Spots
- **Location**: Phase 2 §1
- **Detail**: `postgres` has BYPASSRLS, so a `public` view without `security_invoker` leaks every user's rows.
- **Fix**: The guard fails on views without `security_invoker=true` and on materialized views granted to anon or authenticated; rule added to the conventions.
- **Decision**: FIXED

### F6 — `--db-url` is deprecated in CLI 3.2.0

- **Severity**: ℹ️ OBSERVATION
- **Impact**: 🏃 LOW — quick decision; fix is obvious and narrowly scoped
- **Dimension**: Plan Completeness
- **Location**: Phase 4 §1, steps 4.2 and 4.8
- **Detail**: Use `--remote=<url>`. In 4.2 the first run also reports up to date.
- **Fix**: `--remote="$SUPABASE_DB_URL"` everywhere; 4.2 criterion reworded (Progress title unchanged).
- **Decision**: FIXED

### F7 — Stale line references, and docs left out of the update

- **Severity**: ℹ️ OBSERVATION
- **Impact**: 🏃 LOW — quick decision; fix is obvious and narrowly scoped
- **Dimension**: Plan Completeness
- **Location**: Current State Analysis, Key Discoveries, Phase 4 §3
- **Detail**: deployment-plan.md, ci.yml and dashboard.astro refs were off, and infrastructure.md:113-115 and README.md weren't in the doc updates.
- **Fix**: Refs corrected; infrastructure.md and README.md added to Phase 3 §2 and Phase 4 §3.
- **Decision**: FIXED

### F8 — New scripts will hit ESLint `no-undef`

- **Severity**: ℹ️ OBSERVATION
- **Impact**: 🏃 LOW — quick decision; fix is obvious and narrowly scoped
- **Dimension**: Plan Completeness
- **Location**: Phase 1 §3, Phase 3 §1, Phase 4 §1
- **Detail**: `eslint.config.js:76` declares only console, process, fetch and URLSearchParams for scripts.
- **Fix**: The script contracts say to import from `node:*` modules or extend the globals.
- **Decision**: FIXED
