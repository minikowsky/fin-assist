# Plan: First Cloudflare Workers deployment of fin-assist

## Context

`context/foundation/infrastructure.md` picked **Cloudflare Workers** (Static Assets, not Pages). The repo already builds for workerd (`@astrojs/cloudflare` 14.3.1, wrangler 4.131.1, astro 7.3.2, all checked in `node_modules`), but nothing is deployed yet:

- `wrangler.jsonc` still says `"name": "10x-astro-starter"`.
- wrangler isn't logged in.
- There's no hosted Supabase project. `.env` has placeholders under the **wrong names** (`PUBLIC_SUPABASE_*`, while the app reads `SUPABASE_URL` / `SUPABASE_KEY`).

The goal is a working production URL at `https://fin-assist.<subdomain>.workers.dev` with Supabase auth working end to end, a known rollback target, and every edge case found during research handled up front.

Decisions made:
- Create a **new Supabase project in eu-central-1**. *(Actual, 2026-10-24: the project was created in **eu-north-1 / Stockholm**, still in the EU.)*
- Start on the **Workers Free plan**, with a hard gate to move to Paid before CSV import ships.
- **Deploy model:** the first production deploy is **manual** (`wrangler deploy` from the CLI). After that, **auto-deploy on push to `master` is handled by Cloudflare Workers Builds** (Git integration). There's **no GitHub Actions deploy**: the existing `.github/workflows/ci.yml` stays as is (lint, build, local smoke test) and never deploys. A manual deploy stays available as an escape hatch (see Phase 6).

The repo is `github.com/minikowsky/fin-assist`. It's **public**, the default branch is `master`, and `master` has no branch protection.

Who runs each step: 🤖 = agent may run it unattended, 👤 = human runs it or approves it (production mutations and secrets, per CLAUDE.md).

**Tracking:** this plan is stored at `context/changes/deployment/deployment-plan.md`. That file is the live tracker: its checkboxes get ticked, and outcomes (URL, version IDs) are filled in, as each phase completes.

---

## Phase 0: Prerequisites (accounts, CLIs, local Supabase)

Machine state checked on 2026-10-23:

| Tool | State | Action |
|---|---|---|
| Node | v22.14.0 via fnm, matching `.nvmrc`; system Node v24.22.0 is still there (checked 2026-10-24) | done |
| wrangler | 4.131.1, run through `npx`; logged in via OAuth (checked 2026-10-24) | done |
| Supabase CLI | 2.117.0, run through `npx`; logged in, project created and linked (checked 2026-10-24) | done |
| Docker | 29.1.3 installed; user added to `docker` group, local Supabase running (checked 2026-10-24) | done |
| `supabase/.temp` | ownership fixed (`chown` on all of `supabase/`, 2026-10-24) | done |
| gh | logged in as `minikowsky`, scopes `repo` and `workflow`; `ADMIN` on the repo (checked 2026-10-24) | done |

### 0.1 Accounts (👤)

- [x] **Cloudflare:** a free account, with the email verified. *(Account "Minik", ID `8fbf703f2edc4d1a78b14caabc96bb67`.)*
- [x] **Supabase:** an account and an organization on the Free plan. *(Org ID `dtglhywyyftdeqtunjmd`.)*
- [x] **GitHub:** admin rights on `minikowsky/fin-assist`. You need these to install the Cloudflare GitHub App in Phase 6. *(`viewerPermission: ADMIN`.)*
- [x] **Password manager:** add entries for the Supabase DB password, the admin user and the smoke user. None of these values go into the repo or into chat. *(2026-10-24: DB password saved. Admin and smoke user entries added in Phase 2.)*

### 0.2 Node (👤 install, 🤖 verify)

- [x] **Install a version manager.** For example fnm (`curl -fsSL https://fnm.vercel.app/install | bash`, then restart the shell), then `fnm install && fnm use`. This makes local Node match `.nvmrc` and Workers Builds. *(2026-10-24: fnm in `~/.local/share/fnm`, Node v22.14.0 installed. Non-interactive shells, such as the agent's, don't load fnm from `.bashrc` and fall back to the system Node v24.22.0.)*
- [x] **Install dependencies.** Run `node -v` (expect `v22.14.0`), then `npm ci`. *(`node -v` → v22.14.0, npm 10.9.2; `node_modules` present.)*
- **Edge case:** if you skip this, Node 24 still runs wrangler and Astro. Just remember that local Node differs from CI if a "works on my machine" bug shows up.

### 0.3 wrangler CLI (👤 login, 🤖 checks)

- [x] 👤 **Log in.** Run `npx wrangler login`. It opens a browser for OAuth, and the token is stored in `~/.config/.wrangler/config/default.toml`, outside the repo.
- [x] 🤖 **Check the login.** `npx wrangler whoami` should show the account name and account ID. Note the account ID. *(2026-10-24: one account, "Minik", `8fbf703f2edc4d1a78b14caabc96bb67`. The OAuth token has broad default scopes, so use a scoped token for anything unattended.)*
- [x] Optional: turn off usage telemetry with `npx wrangler telemetry disable`. *(2026-10-24: `wrangler telemetry status` shows Disabled.)*

**Edge cases**
- **The browser callback fails** (SSH, WSL, or a remote shell). The OAuth callback goes to `localhost:8976` on the machine running wrangler, so open the URL on that same machine. Otherwise use an API token:
  1. In the dashboard, go to My Profile → API Tokens → Create Token → template **"Edit Cloudflare Workers"**.
  2. Restrict it to this account only, with no zones.
  3. `export CLOUDFLARE_API_TOKEN=… CLOUDFLARE_ACCOUNT_ID=…` in `~/.bashrc` or a gitignored file. Never put it in the repo; it's public.
- **More than one account** shows in `whoami`. Export `CLOUDFLARE_ACCOUNT_ID` so a deploy can't land in the wrong account.
- **A token-auth login is expired or revoked.** `npx wrangler logout`, then log in again.

### 0.4 Supabase CLI (👤 login, 🤖 checks)

- [x] 👤 **Log in.** Run `npx supabase login`. It opens a browser, and the token goes to the OS keyring.
  - Headless fallback: create a personal access token at supabase.com/dashboard/account/tokens and `export SUPABASE_ACCESS_TOKEN=…` (not in the repo).
- [x] 🤖 **Check the login.** `npx supabase orgs list` should list your organization. Note the org ID. *(Checked with `projects list`: org `dtglhywyyftdeqtunjmd`.)*
- [x] **Create the production project.** This replaces the first step of Phase 2. *(2026-10-24: `fin-assist`, ref `ckvcyftxbatipvjhrome`, region **eu-north-1** (not eu-central-1 as planned), Postgres 17, status `ACTIVE_HEALTHY`.)* Use either:
  - **Dashboard:** project `fin-assist`, region **Central EU (Frankfurt) / eu-central-1**, a generated DB password saved to the password manager.
  - **CLI** (👤, because it creates a billable resource): `npx supabase projects create fin-assist --org-id <org-id> --region eu-central-1 --db-password <pw>`.
- [x] 👤 **Link the repo to the project.** Run `npx supabase link --project-ref ckvcyftxbatipvjhrome` and enter the DB password. *(2026-10-24: linked with no password given. `supabase db push` will ask for the DB password later.)* This writes only to the gitignored `supabase/.temp/`. It's needed later for `supabase db push` once migrations exist.
- [x] 🤖 **Check the link.** `npx supabase projects list` should mark `fin-assist` as linked and show region `eu-central-1`. *(`linked: true`, region `eu-north-1`.)*

**Edge cases**
- **Never run `npx supabase config push` against production.** The local `supabase/config.toml` has `site_url = http://127.0.0.1:3000`, `enable_signup = true` and `enable_confirmations = false`. Pushing it would overwrite the production auth settings from Phase 2. Production auth is configured only in the dashboard.
- **`supabase link` fails with a password or connection error.** Reset the DB password in Dashboard → Database → Settings, or retry with `--password`. On IPv4-only networks, the direct connection is IPv6, so the CLI needs to use the pooler (a recent CLI does this automatically).
- **Supabase Free allows 2 active projects per organization.** If the limit is hit, pause or delete an unused project (👤, in the dashboard).

### 0.5 Local Supabase for dev and smoke tests (👤 sudo, 🤖 the rest)

- [x] 👤 **Give your user Docker access.** Run `sudo usermod -aG docker $USER`, then log out and back in (or run `newgrp docker`). Membership of the `docker` group is equivalent to root; rootless Docker is the stricter alternative.
- [x] 👤 **Fix the root-owned folder.** `sudo chown -R $USER:$USER supabase/.temp`. *(Done 2026-10-24 for all of `supabase/`.)*
- [x] 🤖 **Check Docker.** `docker info` runs without `permission denied`. *(2026-10-24: confirmed by the user; the agent's shell predates the group change and still gets `permission denied`.)*
- [x] 🤖 **Start local Supabase.** Run `npx supabase start`. *(2026-10-24: running; `http://127.0.0.1:54321/auth/v1/health` → 200.)*
- [x] 🤖 **Point local env at local Supabase.** Run `npx supabase status -o env`. Copy `API_URL` → `SUPABASE_URL` and `ANON_KEY` → `SUPABASE_KEY` into both `.env` and `.dev.vars`. This feeds the Phase 1 env-name fix. *(2026-10-24 16:00: `.env` done: `http://127.0.0.1:54321` with the local `anon` key, which local Supabase accepts. `.dev.vars` created with the same two keys.)*
- **Edge cases:**
  - **Never run `supabase` with `sudo` again.** That's what created the root-owned `.temp`.
  - **Ports 54321–54324 already in use:** `npx supabase stop --all`, then start again.
  - **First start is slow:** it pulls about 2 GB of images. The CI job skips the unneeded services with `-x studio,imgproxy,…`, and you can do the same locally.

---

## Phase 1: Repo changes (🤖, local, reversible)

- [x] **Rename the Worker.** In `wrangler.jsonc`, change `"name"` from `"10x-astro-starter"` to `"fin-assist"`. Renaming after the first deploy would create a second Worker, so this must happen first. *(Done 2026-10-24, not committed yet.)*
- [x] **Turn off Astro sessions.** Add `session: false` to `astro.config.mjs`. *(Done 2026-10-24; `grep session src/` found nothing. Verification also turned up an `IMAGES` binding from the adapter's default `imageService: "cloudflare-binding"`, so the adapter is now `cloudflare({ imageService: "passthrough" })`. Nothing in `src/` uses `astro:assets`.)*
  - The adapter (`node_modules/@astrojs/cloudflare/dist/index.js:107`) otherwise injects a `SESSION` KV binding that wrangler auto-provisions on deploy. That's an extra resource, and it breaks deploys made with scoped tokens that lack KV permission.
  - Nothing uses `Astro.session`: auth runs on Supabase cookies in `src/lib/supabase.ts`. First confirm with `grep -r "session" src/`.
- [x] **Make the smoke test safe for production.** `scripts/smoke.mjs` always signs up a fresh `smoke-…@example.com` account. Against production that would:
  - create junk users,
  - fail once sign-up is disabled (the PRD says there's no sign-up flow and the admin creates accounts),
  - send to an address Supabase's default mailer can't deliver to (it only delivers to team members).

  Change: when `SMOKE_EMAIL` and `SMOKE_PASSWORD` are set, skip the `signup creates account` step and sign in with those credentials. Without them, the behaviour stays exactly as it is today, so the CI smoke job against local Supabase is unchanged. *(Done 2026-10-24.)*
- [x] **Fix stale guidance.** *(Done 2026-10-24.)*
  - `context/foundation/tech-stack.md`: change `deployment_target: cloudflare-pages` to `cloudflare-workers`.
  - CLAUDE.md "Environment → Deploy": add one line saying Workers only, never `wrangler pages deploy`; always run `npm run build` right before `npx wrangler deploy`, because wrangler deploys whatever is in `dist/` via the generated `.wrangler/deploy/config.json`.
- [x] **Fix the local env names.** In the gitignored `.env`, rename the keys to `SUPABASE_URL` / `SUPABASE_KEY`, and create `.dev.vars` with the same keys. Local dev should point at **local** Supabase (`npx supabase start`), never at production. *(Done 2026-10-24 as part of Phase 0.5.)*
- [x] **Verify:** *(2026-10-24 on Node v22.14.0: lint clean; `astro check` 0 errors/warnings; build OK with no sessions/KV/Images messages; `dist/server/wrangler.json` → `name: fin-assist`, `kv_namespaces: []`, no `images`; `.wrangler/deploy/config.json` present; smoke against local Supabase passes 8/8 in signup mode and 7/7 in `SMOKE_EMAIL`/`SMOKE_PASSWORD` mode.)*
  - `npm run lint && npx astro check && npm run build` passes.
  - `dist/server/wrangler.json` shows `name: fin-assist` and has **no** `kv_namespaces` and no `images` binding.
  - `.wrangler/deploy/config.json` exists.
  - The local smoke run (local Supabase + `npm run preview` + `npm run smoke`, the same as CI) passes.
- [x] 👤 **Commit and push to `master`.** *(2026-10-24: `98d834f`.)* The agent prepares the commit and you approve it. Workers Builds builds from GitHub, so the rename and `session: false` must be on `master` before Phase 6. The repo is public, so check `git status` first and make sure no `.env` or `.dev.vars` file is staged.

**Edge cases**
- If the build still reports "Enabling sessions with Cloudflare KV", `session: false` isn't being applied. Check where it sits in the config: top level of `defineConfig`, not inside `adapter`.
- `.nvmrc` pins 22.14, and the local Node is v24. wrangler 4.x runs on both. Run `nvm use` if anything behaves differently from CI.

## Phase 2: Configure production Supabase auth (👤, dashboard)

The project was created and linked in Phase 0.4.

- [x] **Turn off sign-up.** Authentication → Sign In / Providers → turn **off** "Allow new users to sign up". This enforces the PRD's admin-only account creation even though `/auth/signup` still exists in the app.
- [x] **Create the users.** Authentication → Users → Add user, with **Auto Confirm User** ticked:
  - the admin account,
  - a dedicated smoke account, for example `smoke@<your-domain>`. RLS keeps its data isolated.
  - Save both sets of credentials in the password manager (finishes step 0.1). *(Done 2026-10-24, confirmed by the user.)*
- [x] **Copy the credentials.** From Project Settings → API, copy the **Project URL** and the **anon / publishable** key.

**Edge cases**
- **Never use the `service_role` / secret key as `SUPABASE_KEY`.** It bypasses RLS, which would break the PRD rule that no user's data is visible to anyone else.
- Free Supabase projects **pause after about 7 days without activity**. Sign-in then fails with network or 5xx errors. Fix it with Dashboard → Restore project. Note it in the risk register; later, consider a keep-alive or the Pro plan.
- The default mailer is capped at about 2 emails per hour and only delivers to team members. That's fine because confirmation emails aren't needed. Set up custom SMTP only if password reset (FR-015) is ever done by email.

## Phase 3: Cloudflare account readiness (👤 dashboard, 🤖 checks)

wrangler is configured in Phase 0.3.

- [x] 🤖 **Recheck the login.** `npx wrangler whoami` should still show the expected account. *(2026-10-25: OAuth, account "Minik" `8fbf703f2edc4d1a78b14caabc96bb67`.)*
- [x] 👤 **Check the workers.dev subdomain.** In the dashboard, go to Workers & Pages and confirm a `*.workers.dev` subdomain is registered. If none is, register it there. Otherwise the first `wrangler deploy` stops on an interactive prompt. *(2026-10-25: checked via API `GET /accounts/…/workers/subdomain` → `minik`. Target URL: `https://fin-assist.minik.workers.dev`. `wrangler deployments list --name fin-assist` → "Worker does not exist" (code 10007), so this is a clean first deploy.)*

## Phase 4: First deploy and secrets (👤 approves each production mutation)

- [x] 👤 **Deploy:** `npm run build && npx wrangler deploy`. Write down the URL and the version ID. *(2026-10-25, approved by the user, run on Node v22.14.0 from `98d834f`: `https://fin-assist.minik.workers.dev`, version `87a824c3-45a9-4d23-9045-5cd5d29e05b2` (no secrets, **not** a rollback target). Bindings: `ASSETS` only. Startup time 13 ms.)*
- [x] 🤖 **Check the deploy without secrets.** `curl -s https://fin-assist.<sub>.workers.dev/` should return 200 and contain the "Supabase nie jest skonfigurowany" banner. That's expected: the secrets are `optional: true` in `astro.config.mjs`. *(2026-10-25: 200, banner present.)*
- [x] 👤 **Set the secrets.** Run `npx wrangler secret put SUPABASE_URL`, then `npx wrangler secret put SUPABASE_KEY`, pasting the values at the prompt. Each command creates a new version immediately. *(2026-10-25, by the user: versions `06b8facc-758c-4b27-82d1-9eaffbe59475` (first put, one secret only) and `1499a110-54d9-4027-abe3-51a942bb46fe` (second put, both secrets). **First valid rollback target: `1499a110-…`.**)*
- [x] 🤖 **List the secrets.** `npx wrangler secret list` should show both names. *(2026-10-25: `SUPABASE_KEY`, `SUPABASE_URL`, both `secret_text`.)*
- [x] 👤 **Set Supabase URLs.** In Supabase → Authentication → URL Configuration, set the Site URL and Redirect URLs to `https://fin-assist.<sub>.workers.dev`. *(2026-10-25: both set to `https://fin-assist.minik.workers.dev`, by the user.)*

**Edge cases**
- **Deploy before setting secrets.** Running `secret put` on a Worker that doesn't exist yet creates it out of band.
- **Rollback trap.** Version 1 has no secrets, and a rollback doesn't change connected resources. **Never roll back to the version from before the secrets were set.** The first valid rollback target is the version created by the `SUPABASE_KEY` put.
- **Error 1102 / "exceeded resource limits"** means the Free plan's 10 ms CPU limit was hit. Check the CPU time in Workers Logs. If auth pages alone go over it, upgrade to Paid now instead of in Phase 6.
- **`[unenv] … not implemented` in the logs** means a dependency uses a Node API that workerd doesn't polyfill. It happens only in the Worker, so reproduce it with `npm run preview` locally.

## Phase 5: Verification (🤖)

- [x] **Banner gone.** `curl` on the home page returns 200 **without** the not-configured banner. *(2026-10-25: 200, banner absent.)*
- [x] **Production smoke test.** Run `BASE_URL=https://fin-assist.<sub>.workers.dev SMOKE_EMAIL=… SMOKE_PASSWORD=… npm run smoke`. All steps should pass, with signup skipped. *(2026-10-25: run by the user against `https://fin-assist.minik.workers.dev`, all steps passed, signup skipped.)*
- [x] **Live logs.** Run `npx wrangler tail fin-assist --format json` while 👤 signs in through the browser. There should be no exceptions, and the CPU time should be well under 10 ms. *(2026-10-25, tailed during the production smoke run plus 10 probe requests: 18 events, all `outcome: ok`, 0 exceptions, no error/warn logs. CPU **over 10 ms on a fresh isolate**: `/auth/signin` 20–31 ms, `/` 13–16 ms for the first 2–3 hits. Warm requests are 1–5 ms, and the API routes 2–11 ms. Nothing was killed (no 1102 error). Watch for 1102 in Workers Logs; it's another reason to keep the Paid upgrade gate.)*
- [x] **Rollback target.** Run `npx wrangler deployments list` and record the current version ID as the known-good rollback target. *(2026-10-25: `1499a110-54d9-4027-abe3-51a942bb46fe`, 100% traffic.)*
- [x] **Cookies.** In the browser, check that the `sb-*` auth cookies are `Secure` and `HttpOnly`/`SameSite` as set by `@supabase/ssr`, that `/dashboard` redirects after sign-out, and that a hard refresh keeps the session. *(2026-10-25: confirmed by the user.)*

**If verification fails:** use `npx wrangler rollback <known-good-id> --message "reason"` (👤), or fix forward. There are no DB migrations yet, so nothing on the Supabase side needs undoing.

## Phase 6: Auto-deploy on push to `master` with Workers Builds (👤 dashboard, 🤖 checks)

Only connect Builds after Phase 5 passes, so the first auto-deploy lands on a Worker that's already known to be good.

- [x] 👤 **Connect the repo.** In the dashboard: Workers & Pages → `fin-assist` → Settings → Builds → **Connect**. Install or authorize the **Cloudflare Workers & Pages GitHub App**, limited to the `minikowsky/fin-assist` repo only (not all repos), and select the repo. *(2026-10-25: in this dashboard version, "Connect" is the **GitHub** button under "Git repository". The app had first been installed from the GitHub side, so the button only opened GitHub's installation settings and never showed the form. Fix: uninstall the app on GitHub, click the GitHub button in Cloudflare again, then **Install & Authorize** (only `fin-assist`). That redirects back to the build form.)*
- [x] 👤 **Build configuration:** *(2026-10-25: set as below, with the corrected build command.)*
  - **Root directory:** `/`
  - **Build command:** `npx astro sync && npm run lint && npx astro check && npm run build`. *(2026-10-25: the first Builds run failed with 20 `no-unsafe-*` lint errors because `astro sync` was missing. `.astro/` is gitignored, so on a clean checkout `astro:middleware`, `astro:env/server` and `import.meta.env` don't resolve until it runs. CI does the same at `ci.yml:19`.)* This puts the quality gate on Cloudflare's side. Builds runs independently of GitHub Actions CI, and `master` isn't protected, so without the gate a commit that fails lint or the type check would still deploy.
  - **Deploy command:** `npx wrangler deploy` (the default). It uses the repo's pinned wrangler 4.131.1 and the generated `.wrangler/deploy/config.json`.
  - **Build variables:** none. The Supabase values are runtime secrets that already live on the Worker (Phase 4), and `env.schema` marks them optional, so the build doesn't need them. Never put them in build variables.
  - **API token:** the auto-generated one, which covers Workers scripts and KV.
- [x] 👤 **Branch control:** *(2026-10-25: production branch `master`, non-production builds off, confirmed by the user.)*
  - **Production branch:** `master`.
  - **Enable Preview Builds:** **off**. Previews share production secrets, so they would talk to production Supabase, and the default `npx wrangler preview` command is beta.
- [x] 🤖 **Trigger the first auto-deploy.** Push a trivial commit, for example the updated tracker file. Then check: *(2026-10-25: connecting didn't start a build. The push of `fcdf8f9` triggered one, which failed lint because `astro sync` was missing (see Build configuration). The retry with the fixed command **succeeded**: version `0cd08f45-40d1-488d-89d3-fed0192917af`, 100% traffic. wrangler shows its source as "Unknown (deployment)", not "Builds", so Build History is where to confirm it. Both secrets carried over, `/` returns 200 with no banner, and the production smoke test passed (user).)*
  - Build History shows success.
  - `npx wrangler deployments list` shows a new version whose source is Builds.
  - `npx wrangler secret list` still shows both secrets, because deploys keep existing secrets.
  - The production smoke test (Phase 5 command) passes again.
- [x] **Manual deploy, the escape hatch** (👤). Use one of these two: *(Documented only, not exercised.)*
  - **Dashboard:** Build History → **Retry build** on a `master` build. Preferred, because it builds from Git.
  - **CLI:** `npm run build && npx wrangler deploy`. Only from a **clean, up-to-date `master`** (`git status` clean, `git pull` done). Otherwise the next push overwrites it, and production would briefly run code that isn't on `master`.

**Edge cases**
- **"Worker name must match"** build error. `wrangler.jsonc` `name` must be `fin-assist`. That's the reason Phase 1 was pushed before connecting.
- **Node version.** Builds reads `.nvmrc` (`22.14.0`). The build image preinstalls 22.23.2 and 24.18.0. If installing 22.14.0 slows or breaks the build, set the build variable `NODE_VERSION=22`, or change `.nvmrc` to `22` (CI already uses `22`).
- **Build passes but the site returns 404 or a blank page.** The build command must run `astro build`. `assets.directory` is `./dist`, which is correct.
- **Two deploys in a row.** Builds on Free allows 1 concurrent build, 20 min timeout, and 3,000 build minutes a month. Back-to-back pushes queue up, and the last one wins.
- **GitHub App errors after renaming or moving the repo.** Disconnect in Settings → Builds, then reconnect. Removing and reinstalling the GitHub App fixes permission errors.
- **Rollback with auto-deploy on.** `wrangler rollback` fixes production immediately, but the **next push to `master` deploys again**. So revert or fix the bad commit on `master` too, otherwise the problem comes back.
- **Stale or deleted build token.** The build fails on auth. Create a new token and select it in the Builds settings.

## Phase 7: Record and hand off (🤖)

- [x] **Finalize the tracker.** In `context/changes/deployment/deployment-plan.md`, tick every checkbox and fill in the outcomes (see **Outcome** below):
  - the URL,
  - the version IDs and the rollback target,
  - the names of the secrets that are set,
  - the Supabase region and the Cloudflare plan,
  - the Builds settings: build command, production branch, preview builds off.
- [x] **Update CLAUDE.md "CI".** Say that GitHub Actions only lints, builds and runs the smoke test, and that production deploys come from Cloudflare Workers Builds on push to `master`. *(2026-10-25: done; it also notes the `astro sync` requirement and that Builds ships even when CI fails.)*
- [x] **Add follow-up gates** to `deployment-plan.md`. None of them run now. *(Listed below; open items stay unticked under **Follow-up gates**.)*
  - **Upgrade to Workers Paid ($5/month) before CSV import ships.** Required because of the CPU cliff and the 50-subrequest cap. Import code must batch LLM calls and do bulk inserts.
  - Consider branch protection on `master`: require GitHub Actions CI to pass and work through PRs.
  - Before turning on preview builds, set up a separate `fin-assist-preview` Worker plus its own Supabase project, because previews share production secrets.
  - `OPENROUTER_API_KEY`: add it to `env.schema` and `wrangler secret put` it when prediction lands.
  - When the first table lands, run `supabase link` + `supabase db push` for migrations, with RLS enabled.
  - Remove the `/auth/signup` page and endpoint (the PRD has no sign-up flow).
  - Re-check Workers Logs pricing after 2026-12-01.

## Outcome (2026-10-25)

| Item | Value |
|---|---|
| Production URL | `https://fin-assist.minik.workers.dev` |
| Cloudflare | account "Minik" `8fbf703f2edc4d1a78b14caabc96bb67`, **Workers Free** plan, Worker `fin-assist` |
| Supabase | project `fin-assist`, ref `ckvcyftxbatipvjhrome`, **eu-north-1** (Stockholm), Free plan, sign-up off |
| Worker secrets | `SUPABASE_URL`, `SUPABASE_KEY` (anon/publishable key) |
| Bindings | `ASSETS` only (no KV, no Images) |
| Versions | `87a824c3-…` first deploy, no secrets (**never roll back to it**) → `06b8facc-…` one secret → `1499a110-54d9-4027-abe3-51a942bb46fe` both secrets, manual → `0cd08f45-40d1-488d-89d3-fed0192917af` first Workers Builds deploy |
| **Rollback target** | `0cd08f45-40d1-488d-89d3-fed0192917af` (current; previous known-good `1499a110-…`) |
| Workers Builds | repo `minikowsky/fin-assist`, root `/`, build `npx astro sync && npm run lint && npx astro check && npm run build`, deploy `npx wrangler deploy`, production branch `master`, preview builds **off**, no build variables |
| Supabase auth URLs | Site URL and Redirect URL `https://fin-assist.minik.workers.dev` |

## Follow-up gates (open, none run now)

- [ ] **Upgrade to Workers Paid ($5/month) before CSV import ships.** Fresh-isolate requests already take 13–31 ms of CPU against the 10 ms Free limit (Phase 5); watch Workers Logs for error 1102 until then.
- [ ] Branch protection on `master`, requiring GitHub Actions CI to pass and work to go through PRs. Builds deploys every push regardless of CI.
- [ ] Before turning on preview builds, set up a separate `fin-assist-preview` Worker and its own Supabase project.
- [ ] `OPENROUTER_API_KEY`: add it to `env.schema` and `wrangler secret put` it when prediction lands.
- [ ] First table: `supabase db push` for migrations, with RLS enabled.
- [ ] Remove the `/auth/signup` page and endpoint.
- [ ] Re-check Workers Logs pricing after 2026-12-01.
- [ ] Supabase Free pauses after about 7 days of inactivity: consider a keep-alive or the Pro plan.

## Critical files

- `wrangler.jsonc`: the name
- `astro.config.mjs`: `session: false`, `imageService: "passthrough"`
- `scripts/smoke.mjs`: production-safe credential mode
- `context/foundation/tech-stack.md`: `deployment_target`
- `CLAUDE.md`: the deploy line
- `.env` / `.dev.vars`: local only, gitignored
- `context/changes/deployment/deployment-plan.md`: new, the live tracker for this plan
