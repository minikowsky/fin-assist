---
project: "fin-assist"
context_type: greenfield
created: 2026-10-11
updated: 2026-10-11
product_type: web-app
target_scale:
  users: small
timeline_budget:
  mvp_weeks: 3
  hard_deadline: null
  after_hours_only: true
checkpoint:
  current_phase: 8
  phases_completed: [1, 2, 3, 4, 5, 6, 7]
  gray_areas_resolved:
    - topic: "context type"
      decision: "greenfield — repo is the unmodified Astro starter scaffold; no existing system to preserve"
    - topic: "insight"
      decision: "bank's built-in categories don't fit and can't be customized; no known tool combines user-defined rules with AI suggestions + correction"
    - topic: "primary persona scope"
      decision: "each family member separately, analysing their own expenses"
    - topic: "auth strategy"
      decision: "login + password; no self-registration — accounts created manually by an admin"
    - topic: "roles"
      decision: "admin (dedicated account, only creates user accounts, no expense data, no access to users' data) + regular users (flat, equal)"
    - topic: "data visibility"
      decision: "fully private — each user sees only their own data"
    - topic: "MVP first flow"
      decision: "login → empty dashboard → import CSV → review predicted categories → correct → name & save set → set listed on dashboard; Stats + Account (categories, rules) alongside; 3 weeks accepted"
    - topic: "starting categories"
      decision: "every new account gets a default category list the user can edit"
    - topic: "prediction accuracy metric"
      decision: "share of predicted categories kept unchanged by the user in a saved set"
    - topic: "MVP capability additions"
      decision: "added open/edit saved set, delete saved set, log out, change own password"
    - topic: "rules priority"
      decision: "initially demoted to nice-to-have; promoted back to must-have in the challenge round (rules + AI is the differentiator)"
    - topic: "login identifier"
      decision: "email address + password"
    - topic: "category rename/remove"
      decision: "rename propagates to all records; records in a removed category revert to '?'"
    - topic: "non-expense records"
      decision: "income, refunds, own-account transfers get special categories excluded from spending charts"
    - topic: "stats period"
      decision: "user selects one or more saved sets"
    - topic: "domain rule precedence"
      decision: "user rule first, else predicted category, else '?'"
    - topic: "rule matching"
      decision: "rule text appears anywhere in title or description, case-insensitive"
    - topic: "rule conflicts"
      decision: "record matched by rules pointing to different categories is marked '?'"
    - topic: "prediction confidence"
      decision: "prediction shows '?' when not confident"
    - topic: "learning from corrections"
      decision: "no — predictions don't learn; rules are the user's way to make a correction stick"
    - topic: "privacy of predictions"
      decision: "only title/description leave the app for prediction; never user identity; users are informed"
    - topic: "import/prediction speed"
      decision: "no target"
    - topic: "scale"
      decision: "just the user and a handful of family members; domain rule would not change at 100x"
    - topic: "timing"
      decision: "no hard deadline; after-hours work"
    - topic: "non-goals"
      decision: "4 from idea notes + no bank connection, no budgets/alerts, no export, no multi-currency"
  frs_drafted: 14
  quality_check_status: accepted
---

# Shape notes

Seed: `draft/idea-notes.md`

## Vision & Problem Statement

Categorizing expenses is slow: every transaction has to be read and labelled by hand. Knowing how much goes to each category is essential for managing a home budget. The user and their family (all customers of one particular bank) do this by hand whenever they analyse their expenses (weekly or monthly, depending on the person). It costs about one hour per month, and sometimes they give up on it entirely, which leaves them with no picture of where the money goes.

Insight: the bank's built-in categories don't match how the family thinks about spending, are wrong too often, and can't be customized. No tool the user knows combines categorization rules the user writes ("contains LIDL → Shopping") with AI-suggested labels for everything else that the user can correct.

## User & Persona

**Primary persona:** A family member who is a customer of the bank and periodically (weekly or monthly) analyses their own expenses. They reach for the product when they sit down to review a period's spending and want it broken down by category without labelling every transaction by hand. Each family member works with their own data separately.

Scale: the user and a handful of family members. At 100× that scale the domain rule would not change (per user).

## Success Criteria

MVP first flow (first session):

1. User logs in → dashboard with an empty list of expense sets (header: Account · Import CSV · Stats).
2. (optional) Account → manage their own category list and categorization rules.
3. Import CSV → upload the bank's CSV file.
4. Review screen: each record shows date, title, amount, description, and a predicted category (`?` when there is no prediction).
5. User corrects wrong categories, or picks one from the list for any `?`.
6. User names the set (e.g. "May 2026") and saves it.
7. Redirect to dashboard → the named set appears in the list.
8. Stats → charts of spending per category over a period.

### Primary
- A user can import their bank's CSV file and end up with a saved, named, fully categorized set of expenses (the flow above works end to end).
- ≥ 75% of predicted categories in a saved set are kept unchanged by the user.

### Secondary
- Categorizing a month's expenses takes much less time than the ~1 hour it takes by hand today (exact target TBD — see Open Questions).

### Guardrails
- No user's financial data is ever visible to another user or to the admin.
- Import is exact: no record is dropped, duplicated, or altered (amount, date, title).
- A user's own rule or manual category choice is never overridden by a predicted category.

Timeline budget: `mvp_weeks: 3` (within the 3-week default; no extended-timeline acknowledgment needed).

## User Stories

### US-01: User categorizes a month of expenses from a bank CSV

- **Given** a logged-in user with an empty dashboard
- **When** they import their bank's CSV file, correct the predicted categories, name the set "May 2026", and save it
- **Then** they are back on the dashboard and "May 2026" appears in their list of sets

#### Acceptance Criteria
- Every record from the CSV file is shown on the review screen.
- The set cannot be saved while any record still has the `?` category.
- A file not in the supported bank format is rejected with an error.
- A set name is required and must be unique among that user's sets.

## Functional Requirements

### Accounts & access
- FR-001: Admin can create a user account (email + password). Priority: must-have
  > Socratic: Considered "an admin UI is overkill for a few family members" and "a dedicated admin account adds friction". Resolution: no counter-argument; stands as written.
- FR-002: User can log in with email address and password. Priority: must-have
  > Socratic: Counter-argument accepted: "it's unclear whether the login is a username or an email, which affects any future password recovery". Resolution: login is an email address.
- FR-003: User can log out. Priority: must-have
  > Socratic: Considered "session expiry makes log-out cosmetic". Resolution: no counter-argument; stands (shared family devices).
- FR-004: User can change their own password. Priority: nice-to-have
  > Socratic: Counter-argument accepted: "not needed for a family MVP — the admin sets the password once". Resolution: demoted from must-have to nice-to-have.

### Categories & rules
- FR-005: User can manage their own category list, which starts from a default list (add / rename / remove). Renaming a category updates every record that uses it; records in a removed category revert to `?`. Priority: must-have
  > Socratic: Counter-argument accepted: "removing or renaming a category already used by saved sets has undefined effects on those records and on Stats". Resolution: rename propagates; removal reverts affected records to `?`.
- FR-006: User can define a rule mapping record text to a category (e.g. contains "LIDL" → Shopping). Priority: must-have
  > Socratic: Counter-argument accepted: "rules + AI suggestions together are the differentiator; without rules the MVP is AI guesses + manual fixes, the half the bank already does badly". Resolution: promoted from nice-to-have to must-have.

### Import & review
- FR-007: User can import a CSV file in the one supported bank format. Priority: must-have
  > Socratic: Considered "overlapping imports create duplicate transactions across sets" and "the bank may change its CSV layout". Resolution: no counter-argument; stands as written.
- FR-008: User can see the imported records (date, title, amount, description), each with a predicted category or `?` when there is none. Income, refunds, and transfers between own accounts get special categories (e.g. "Income", "Transfer") that are excluded from spending charts. Priority: must-have
  > Socratic: Counter-argument accepted: "a bank CSV contains income, refunds, and own-account transfers; giving them spending categories distorts Stats". Resolution: special categories excluded from spending charts.
- FR-009: User can change any record's category to one from their list. Priority: must-have
  > Socratic: Considered "corrections aren't remembered across imports" and "one-at-a-time changes are slow for long months". Resolution: no counter-argument; stands as written.

### Expense sets
- FR-010: User can name an imported set and save it. Priority: must-have
  > Socratic: Considered "sets duplicate what a date filter does" and "unsaved corrections are lost if the user leaves before saving". Resolution: no counter-argument; stands as written.
- FR-011: User can see the list of their saved sets on the dashboard. Priority: must-have
  > Socratic: Considered "a name alone doesn't reveal date range or totals". Resolution: no counter-argument; stands as written.
- FR-012: User can open a saved set to view it and edit its categories. Priority: must-have
  > Socratic: Considered "editing an old set silently changes historical Stats" and "it duplicates the import review screen". Resolution: no counter-argument; stands as written.
- FR-013: User can delete a saved set. Priority: must-have
  > Socratic: Considered "deletion is irreversible and loses all manual corrections". Resolution: no counter-argument; stands as written.

### Stats
- FR-014: User can see charts of spending per category for one or more saved sets they select. Priority: must-have
  > Socratic: Counter-argument accepted: "it's unclear whether 'a given period' means a date range or chosen sets". Resolution: the user selects one or more saved sets.

## Non-Functional Requirements

- Only the record content needed to predict a category (title, description) ever leaves the product for prediction; the user's identity and credentials never do. Users are told that this happens.
- A deleted set's records are permanently gone and cannot be seen anywhere in the product afterwards.
- The product works on the current versions of the four mainstream desktop browsers (Chrome, Firefox, Edge, Safari).
- The user interface is in Polish.
- No response-time target for import and prediction (explicitly chosen: a weekly/monthly task).

## Business Logic

For each imported record, a matching user-defined rule decides its category; if no rule matches, a predicted category is proposed; if neither yields a category, the record is marked `?`.

**Inputs:** the record's date, title, amount, and description from the bank CSV, plus the user's own category list and rules. A rule matches when its text appears anywhere in the record's title or description (case-insensitive). If several of the user's rules match one record and point to different categories, the record is marked `?`; no prediction is applied.

**Output:** one category per record. Rule matches take precedence over predictions. A prediction that isn't confident yields `?` rather than a guess. Income, refunds, and own-account transfers fall into special categories that are excluded from spending charts.

**How the user meets it:** right after import, on the review screen, every record already shows a category (or `?`). The user corrects what's wrong and resolves every `?` before saving. Predictions don't learn from past corrections; to make a correction stick for future imports, the user writes a rule.

## Access Control

- **Authentication:** login with password. No self-registration.
- **Roles:**
  - **Admin:** a dedicated account used only for managing users. Can create user accounts. Has no expense data of its own and **cannot** see any user's transactions, categories, rules, or stats.
  - **User:** a regular account (flat; all users equal). Has full control over their own imported sets, categories, categorization rules, and stats.
- **Visibility:** fully private. A user sees only their own data; nothing is shared between users.
- **Sign-up vs sign-in:** there is no sign-up flow; the admin creates accounts, and users only sign in.
- Not in admin scope (by choice): deactivating/deleting accounts, resetting passwords.

## Non-Goals

Functional:
- **No advanced authentication.** Email + password only (from idea notes).
- **No support for multiple banks' CSV formats.** One known format from one bank (from idea notes).
- **No own label-prediction model.** Predictions come from an outside service; no model is built or trained (from idea notes).
- **No mobile application.** Web application only (from idea notes).
- **No automatic bank connection.** CSV upload is the only way data comes in.
- **No budgets, limits, or alerts.** The MVP describes spending; it doesn't police it.
- **No data export.** Categorized sets and stats are viewed in the app only.
- **No multi-currency.** All amounts are in one currency; no conversion.
- **No self-registration, no admin password reset, no account deactivation/deletion.** The admin only creates accounts (Access Control).
- **No sharing between users.** Every user's data is fully private (Access Control).
- **Predictions don't learn from past corrections.** Rules are the user's way to make a correction stick (Business Logic).
- **User password change is out of MVP scope** (FR-004, nice-to-have).

Non-functional:
- **No response-time target for import and prediction.** It's a weekly or monthly task.
- **No mobile-browser support commitment.** Only desktop browsers are committed to (NFRs).

## Forward: tech-stack

Informational only, not part of the PRD. Observed in the repo, not decided during shaping: the repository is already scaffolded with Astro + React, Tailwind CSS v4, a Cloudflare Workers adapter, Supabase, and the Vercel AI SDK (Anthropic and Google providers). Tech-stack selection should confirm or revise this.

## Open Questions

1. **How does a user who forgets their password regain access?** The admin can only create accounts (no password reset), and self-service reset wasn't discussed. Owner: user.
2. **What happens when an unauthenticated visitor hits a gated page?** Not discussed yet. Owner: user.
3. **What is the target time for categorizing a month (Secondary criterion)?** "Much less than ~1 hour"; a concrete number (e.g. 10 min) is not yet confirmed. Owner: user.
4. **What happens to Stats when selected sets overlap in dates?** Stats covers user-selected sets (FR-014), and overlapping imports (e.g. a weekly set inside a monthly one) would double-count transactions. Not resolved. Owner: user.
5. **How does a saved set behave when a category removal reverts some of its records to `?`?** US-01 blocks saving a set that has `?` records, and FR-005 can create them after saving. Not resolved. Owner: user.
6. **What confidence level makes a prediction show `?` instead of a category?** Agreed that low-confidence predictions yield `?`; the threshold isn't set. Owner: user (tune against the ≥ 75% kept-unchanged target).

## Quality cross-check

Run on 2026-10-11. All required elements present (Access Control, Business Logic, project artifacts, timeline-cost acknowledgment, Non-Goals; preserved behavior n/a for greenfield). No gaps. The open questions above carry forward to the PRD.
