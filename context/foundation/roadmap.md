---
project: fin-assist
version: 1
status: draft
created: 2026-11-02
updated: 2026-11-02
prd_version: 1
main_goal: speed
top_blocker: time
milestone_id: first-categorized-month
milestone_seq: 1
milestone_status: open
---

# Roadmap: fin-assist

> Derived from `context/foundation/prd.md` (v1) + auto-researched codebase baseline.
> Edit-in-place; the next milestone continues this file. Archive only when an independent roadmap replaces it.
> Slices below are listed in dependency order. The "At a glance" table is the index.

## Milestone

**M-1: First categorized month** — Status: open

- **Intent:** A family member can sign up, import a month of their bank's CSV, get it categorized by their own rules and AI suggestions, fix what's wrong, save it as a named set, and see where the money went, with their data visible to no one else.
- **Source materials:** `context/foundation/prd.md` (v1)
- **Done when:** every F-NN and S-NN below is `done`.
- **Scope anchors:** US-01; FR-001–FR-003, FR-005–FR-014 (all must-haves). FR-004 (nice-to-have) is parked.

## Vision recap

Categorizing expenses by hand costs the user and their family about an hour a month, and sometimes they give up and lose sight of where the money goes. The bank's built-in categories don't match how the family thinks about spending and can't be customized. fin-assist combines rules the user writes ("contains LIDL → Shopping") with AI-suggested categories for everything else, which the user can correct before saving.

## North star

**S-03: User can save a reviewed import as a named set and see it on the dashboard**: together with S-02, this completes the PRD's Primary success criterion (bank CSV in → saved, named, fully categorized set out). With `speed` as the goal, nothing else matters until this works.

> The "north star" is the smallest end-to-end slice whose delivery proves the product works. It is placed as early as its prerequisites allow, because everything else only matters if it works. Here it is built with categories picked by hand. Rules (S-04) and AI predictions (S-05) then make the same flow fast.

## At a glance

| ID   | Issue | Change ID                   | Outcome (user can …)                                                                      | Prerequisites | PRD refs                                | Status   |
| ---- | ----- | --------------------------- | ----------------------------------------------------------------------------------------- | ------------- | --------------------------------------- | -------- |
| F-01 | #1    | user-data-isolation         | (foundation) migrations reach production safely; a two-user isolation check exists      | —             | Success Criteria §Guardrails, Access Control §Visibility | ready    |
| S-01 | #2    | open-signup-and-return      | sign up in production, confirm email, log in back to the requested page, log out          | —             | FR-001, FR-002, FR-003, Access Control, NFR (Polish UI) | ready    |
| S-02 | #3    | csv-import-review           | upload the bank CSV and review/edit every record, picking categories from a default list  | —             | US-01, FR-007, FR-008, FR-009           | ready    |
| S-03 | #4    | first-saved-expense-set     | name and save the reviewed set and see it in the dashboard list                           | F-01, S-02    | US-01, FR-010, FR-011                   | proposed |
| S-04 | #5    | categorization-rules        | write text → category rules that pre-fill matching records on the next import             | F-01, S-02    | US-01, FR-006                           | proposed |
| S-05 | #6    | ai-category-predictions     | see AI-predicted categories on records no rule matched (`?` when not confident)           | S-04          | US-01, FR-008, NFR (prediction privacy) | proposed |
| S-06 | #7    | manage-category-list        | add, rename and remove their own categories for future imports                            | F-01, S-02    | FR-005                                  | proposed |
| S-07 | #8    | set-spending-stats          | see a per-category spending chart, category totals and total spending for a chosen set    | S-03          | FR-014                                  | proposed |
| S-08 | #9    | edit-saved-set              | open a saved set and edit any record's data and category                                  | S-03          | FR-012                                  | proposed |
| S-09 | #10   | delete-saved-set            | delete a saved set permanently                                                            | S-03          | FR-013, NFR (permanent deletion)        | proposed |

## Streams

Navigation aid: groups items that share a Prerequisites chain. The canonical order is the dependency graph below; this table shows a suggested reading order across tracks that can run in parallel.

| Stream | Theme                   | Chain                                     | Note                                                                                   |
| ------ | ----------------------- | ----------------------------------------- | -------------------------------------------------------------------------------------- |
| A      | Account access          | `S-01`                                    | Standalone; closes the production sign-up gap so family members can onboard.           |
| B      | Import & categorization | `S-02` → `S-04` → `S-05`                  | The speed-critical path; S-04 also needs `F-01` (Stream C).                            |
| C      | Saved sets & stats      | `F-01` → `S-03` → `S-07` → `S-08` → `S-09` | S-03 joins Stream B at `S-02`; S-07, S-08, S-09 can run in parallel after S-03.        |
| D      | Category list           | `S-06`                                    | Needs `F-01` and `S-02`; then parallel with everything else.                           |

## Baseline

What's already in place in the codebase as of `2026-11-02` (auto-researched + user-confirmed).
Foundations below assume these are present and do NOT re-scaffold them.

- **Frontend:** present: Astro SSR + React islands, Tailwind, shadcn/ui; placeholder dashboard (`src/pages/dashboard.astro`).
- **Backend / API:** partial: only auth API routes (`src/pages/api/auth/*`); no domain endpoints, no input-validation library installed.
- **Data:** absent: Supabase client wired (`src/lib/supabase.ts`), but no migrations, tables or row-level security yet.
- **Auth:** partial: sign-up / sign-in / sign-out / confirm-email pages and endpoints exist, and middleware protects `/dashboard`. Sign-up is turned off in production Supabase (set before the PRD's 2026-10-30 open-sign-up revision), and sign-in always redirects to `/` instead of the requested page.
- **Deploy / infra:** present: Cloudflare Workers at `fin-assist.minik.workers.dev`, auto-deploy on push to `master`, GitHub Actions CI + smoke test. Open gate: Workers Free plan (10 ms CPU) must be upgraded before CSV import ships (`context/changes/deployment/deployment-plan.md`).
- **Observability:** partial: Workers Logs enabled; no app-level logging or error tracking.
- **AI prediction:** absent: no AI SDK or prediction provider in dependencies (tech-stack.md says otherwise; see Open Roadmap Questions).

## Foundations

### F-01: User data isolation and production migration path

- **Outcome:** (foundation) database changes reach production Supabase through a repeatable path, the convention "every user-owned table ships with per-operation row-level security" is in place, and the smoke test can sign in two separate users so that every data slice can assert one user never sees the other's data.
- **Change ID:** user-data-isolation
- **PRD refs:** Success Criteria §Guardrails, Access Control §Visibility
- **Unlocks:** S-03, S-04, S-06 (first slices that store user data); the cross-user isolation check reused by S-07, S-08, S-09.
- **Prerequisites:** —
- **Parallel with:** S-01, S-02
- **Blockers:** —
- **Unknowns:** —
- **Risk:** Kept deliberately thin. It creates no domain tables, since each slice adds its own. Skipping it would mean the first saved financial data reaches production with no proof of isolation, and that guardrail cannot be fixed after a leak.
- **Status:** ready

## Slices

### S-01: Open sign-up and return to the requested page

- **Outcome:** user can sign up in production with email and password, confirm their email, log in and land on the page they originally requested, and log out.
- **Change ID:** open-signup-and-return
- **PRD refs:** FR-001, FR-002, FR-003, Access Control, NFR (Polish UI)
- **Prerequisites:** —
- **Parallel with:** F-01, S-02 (independent of every other item)
- **Blockers:** —
- **Unknowns:**
  - The deployment plan's follow-up "Remove the `/auth/signup` page and endpoint" and the production "sign-up off" setting predate the open-sign-up revision. Confirm both are void. Owner: user. Block: no.
- **Risk:** Most of the code exists; the risk is a production setting (sign-up toggle, email-confirmation redirect URLs) that the smoke test doesn't cover, so family members can't onboard.
- **Status:** ready

### S-02: Import a bank CSV and review every record

- **Outcome:** user can upload their bank's CSV file, see every record (date, title, amount, description) on a review screen with `?` as its category, edit any record's data, and pick a category for each from a default list that includes special categories (income, transfer). A file in any other format is rejected with an error.
- **Change ID:** csv-import-review
- **PRD refs:** US-01, FR-007, FR-008, FR-009
- **Prerequisites:** a real (anonymized) sample of the bank's CSV export in hand
- **Parallel with:** F-01, S-01
- **Blockers:** Upgrade to Workers Paid before this ships to production (10 ms CPU limit on Free; deployment-plan follow-up). This doesn't block planning or local work.
- **Unknowns:**
  - Exact CSV format: encoding, delimiter, columns, and how income, refunds and own-account transfers can be told apart. Owner: user (provides the sample). Block: no.
- **Risk:** Highest-uncertainty slice on the north-star path. The "import is exact" guardrail (no record dropped, duplicated or altered) has to hold for real bank exports, not hand-made fixtures.
- **Status:** ready

### S-03: Save a named set and see it on the dashboard

- **Outcome:** user can name the reviewed set (required, unique among their sets), save it once no record has `?`, return to the dashboard, and see the set in their list.
- **Change ID:** first-saved-expense-set
- **PRD refs:** US-01, FR-010, FR-011
- **Prerequisites:** F-01, S-02
- **Parallel with:** S-04, S-06
- **Blockers:** —
- **Unknowns:** —
- **Risk:** The first slice that writes financial data. It has to pass the F-01 isolation check, and the saved records must equal the reviewed ones exactly.
- **Status:** proposed

### S-04: Categorization rules pre-fill matching records

- **Outcome:** user can define rules in Account (record text → category) and, on the next import, see matching records already categorized. A rule matches case-insensitively anywhere in the title or description, and conflicting matches leave the record `?`.
- **Change ID:** categorization-rules
- **PRD refs:** US-01, FR-006
- **Prerequisites:** F-01, S-02
- **Parallel with:** S-03, S-06
- **Blockers:** —
- **Unknowns:** —
- **Risk:** Sequenced before S-05 so that the "rules always beat predictions" order is set once. S-05 then plugs into it rather than the two slices racing to define it.
- **Status:** proposed

### S-05: AI-predicted categories for unmatched records

- **Outcome:** user sees a predicted category on every record no rule matched, `?` when the prediction isn't confident, and income/refunds/transfers in special categories. The user is told that record titles and descriptions are sent for prediction.
- **Change ID:** ai-category-predictions
- **PRD refs:** US-01, FR-008, NFR (prediction privacy)
- **Prerequisites:** S-04
- **Parallel with:** S-03, S-06, S-07, S-08, S-09
- **Blockers:** A prediction-provider account and API key stored as a production secret; Workers Paid (the subrequest and CPU caps on Free).
- **Unknowns:**
  - Which provider path: OpenRouter (infrastructure.md) or a direct AI SDK provider (tech-stack.md)? Neither is installed. Owner: user. Block: no.
  - Confidence threshold for `?` vs. a guess (PRD: "set during implementation"). Owner: team. Block: no.
- **Risk:** The least predictable slice (prediction quality, per-import call limits on Workers). Predictions must be batched per import, and a rule or manual choice must never be overridden.
- **Status:** proposed

### S-06: Manage the category list

- **Outcome:** user can add, rename and remove categories in their own list, which starts from the default list and is used for new imports. Records in saved sets keep their categories.
- **Change ID:** manage-category-list
- **PRD refs:** FR-005
- **Prerequisites:** F-01, S-02
- **Parallel with:** S-03, S-04, S-05, S-07, S-08, S-09
- **Blockers:** —
- **Unknowns:** —
- **Risk:** Renames and removals must not leak into saved sets or into rules that point at a removed category. The exact behavior for rules is left to `/10x-plan`.
- **Status:** proposed

### S-07: Spending stats for a chosen set

- **Outcome:** user can see on the dashboard, for the set selected in the list (newest by default), a chart of spending per category, a list of spending categories with their totals, and the set's total spending. Special categories get no totals.
- **Change ID:** set-spending-stats
- **PRD refs:** FR-014
- **Prerequisites:** S-03
- **Parallel with:** S-04, S-05, S-06, S-08, S-09
- **Blockers:** —
- **Unknowns:** —
- **Risk:** Low. Its only correctness trap is excluding special categories from totals.
- **Status:** proposed

### S-08: Open and edit a saved set

- **Outcome:** user can open a saved set and edit any record's date, title, amount, description and category (chosen from their current list plus categories already used in that set).
- **Change ID:** edit-saved-set
- **PRD refs:** FR-012
- **Prerequisites:** S-03
- **Parallel with:** S-04, S-05, S-06, S-07, S-09
- **Blockers:** —
- **Unknowns:** —
- **Risk:** Overlaps the S-02 review screen. Plan to reuse it rather than build a second editor.
- **Status:** proposed

### S-09: Delete a saved set

- **Outcome:** user can delete a saved set, and its records are permanently gone from everywhere in the product.
- **Change ID:** delete-saved-set
- **PRD refs:** FR-013, NFR (permanent deletion)
- **Prerequisites:** S-03
- **Parallel with:** S-04, S-05, S-06, S-07, S-08
- **Blockers:** —
- **Unknowns:** —
- **Risk:** Irreversible. It needs a confirmation step, and deleted records must not linger in stats.
- **Status:** proposed

## Backlog Handoff

| Roadmap ID | Issue | Change ID               | Suggested issue title                                         | Ready for `/10x-plan` | Notes                                       |
| ---------- | ----- | ----------------------- | ------------------------------------------------------------- | --------------------- | ------------------------------------------- |
| F-01       | #1    | user-data-isolation     | Production migration path + two-user data isolation check     | yes                   | Run `/10x-plan user-data-isolation`         |
| S-01       | #2    | open-signup-and-return  | Open sign-up in production; return to requested page on login | yes                   | Run `/10x-plan open-signup-and-return`      |
| S-02       | #3    | csv-import-review       | Import bank CSV and review/edit every record                  | yes                   | Have the bank CSV sample ready              |
| S-03       | #4    | first-saved-expense-set | Save a named expense set and list it on the dashboard         | no                    | After F-01, S-02                            |
| S-04       | #5    | categorization-rules    | User-defined categorization rules applied on import           | no                    | After F-01, S-02                            |
| S-05       | #6    | ai-category-predictions | AI-predicted categories for unmatched records                 | no                    | After S-04; needs provider key              |
| S-06       | #7    | manage-category-list    | Manage own category list                                      | no                    | After F-01, S-02                            |
| S-07       | #8    | set-spending-stats      | Per-category spending chart and totals for a set              | no                    | After S-03                                  |
| S-08       | #9    | edit-saved-set          | Open and edit a saved set                                     | no                    | After S-03                                  |
| S-09       | #10   | delete-saved-set        | Delete a saved set permanently                                | no                    | After S-03                                  |

## Open Roadmap Questions

1. **What are the request-rate and data-volume ballparks (`target_scale.qps`, `target_scale.data_volume`)?** (PRD Open Question 1.) Owner: user. Block: none. Issue: #11.
2. **Which prediction-provider path: OpenRouter or a direct AI SDK provider?** tech-stack.md calls the AI SDK "already installed" and infrastructure.md plans an OpenRouter key, but neither is in the dependencies. Owner: user. Block: S-05 (planning can proceed; resolve during its plan). Issue: #12.
3. **Supabase Free pauses a project after about 7 days of inactivity, and this product is used weekly or monthly.** Keep-alive or Pro plan? Owner: user. Block: roadmap-wide (affects real family use, not building). Issue: #13.

## Parked

- **FR-004: User can change their own password.** Why parked: nice-to-have; PRD §Non-Goals ("User password change is out of MVP scope").
- **Advanced authentication.** Why parked: PRD §Non-Goals, email + password only.
- **Multiple banks' CSV formats.** Why parked: PRD §Non-Goals, one known format.
- **Own label-prediction model.** Why parked: PRD §Non-Goals, outside service only.
- **Mobile application / mobile-browser support.** Why parked: PRD §Non-Goals, desktop web only.
- **Automatic bank connection.** Why parked: PRD §Non-Goals, CSV upload only.
- **Budgets, limits, alerts.** Why parked: PRD §Non-Goals.
- **Data export.** Why parked: PRD §Non-Goals.
- **Multi-currency.** Why parked: PRD §Non-Goals.
- **Admin role, in-app password reset, account deactivation/deletion.** Why parked: PRD §Non-Goals / Access Control.
- **Sharing between users.** Why parked: PRD §Non-Goals, fully private.
- **Predictions learning from past corrections.** Why parked: PRD §Non-Goals, rules make corrections stick.
- **Charts across several sets or a date range.** Why parked: PRD §Non-Goals, one set per chart.
- **Response-time target for import and prediction.** Why parked: PRD §Non-Goals, weekly or monthly task.

## Milestone History

## Done
