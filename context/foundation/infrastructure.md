---
project: fin-assist
researched_at: 2026-10-23
recommended_platform: Cloudflare Workers
runner_up: Vercel
context_type: mvp
tech_stack:
  language: TypeScript
  framework: Astro 7 (SSR, React 19 islands) + Supabase + OpenRouter (Vercel AI SDK)
  runtime: Cloudflare workerd (@astrojs/cloudflare 14.3.1, wrangler 4.131.1)
---

## Recommendation

**Deploy on Cloudflare Workers** (Workers with Static Assets, not Pages).

The repo already targets it: `astro.config.mjs` uses `@astrojs/cloudflare`, `wrangler.jsonc` exists, and `astro dev` / `astro preview` already run in workerd, so the first deploy needs no migration. The developer already knows Cloudflare, which broke a three-way near-tie with Vercel and Render on the agent-friendly criteria. Interview answers favoured no other platform strongly: request/response only, cost and DX weighted equally, a single region (Poland), and Supabase + OpenRouter as external services. At family scale (10k–100k requests/month) the cost is $0 on Free or a flat $5 on Workers Paid.

## Platform Comparison

Research run 2026-10-23 with web sources; feature status as of that date.

| Platform | CLI-first | Managed/Serverless | Agent-readable docs | Stable deploy API | MCP / Integration | Score* | Migration from current repo |
|---|---|---|---|---|---|---|---|
| **Cloudflare Workers** | Pass | Pass | Pass | Partial | Pass | 4.5 | None |
| **Vercel** | Pass | Pass | Pass | Pass | Partial (beta) | 4.5 | Low (adapter swap) |
| **Render** | Pass | Partial | Pass | Pass | Pass (GA) | 4.5 | Low (`@astrojs/node`) |
| Netlify | Partial | Pass | Pass | Pass | Pass | 4.5 → weighted down | Low (adapter swap) |
| Fly.io | Pass | Partial | Pass | Pass | Partial (experimental) | 4.0 | Medium (Dockerfile) |
| Railway | Partial | Pass | Pass | Partial | Partial (preview) | 3.5 | Low (`@astrojs/node`) |

\*Pass = 1, Partial = 0.5, Fail = 0. Before soft weights. No platform failed a hard filter: no persistent connections are needed, and every platform runs Node/JS. Only Cloudflare runs the stack as currently configured. The others need an adapter change.

- **Cloudflare Workers.** The `wrangler` CLI covers deploy, versions, rollback, tail and secrets. Docs have a root `llms.txt`, a per-product `llms.txt`, and markdown via `index.md` or `Accept: text/markdown`. The Cloudflare API MCP server and Docs MCP server are current. The Observability, Builds and Bindings MCP servers are **deprecated**, and `/sse` endpoints were retired 2026-10-22. Deploy API is Partial: most wrangler commands lack JSON output, and the `cf` CLI that will replace wrangler is in **open beta** (wrangler gets 18 months of maintenance after cf reaches GA).
  - Free plan: 10 ms CPU and 50 subrequests per request, 100k requests/day.
  - Paid ($5/month): 30 s CPU by default, 10k subrequests, 10M requests included.
  - Time spent waiting on a fetch (LLM calls) does not count as CPU.
- **Vercel.** Fluid compute (Node) is GA and is the default. CLI: `deploy --prod`, `rollback`, `promote`, `logs`. Docs: `llms.txt`, plus a `.md` version of every page. MCP server at `mcp.vercel.com` is in **public beta** (since 2025-08-04, no GA announcement found). Hobby is free but non-commercial (fine for a family app). Hobby limits:
  - 300 s function duration
  - rollback only to the previous production deployment
  - 1 h runtime log retention
  - region pinned via `vercel.json` (`fra1`)

  Migration: `@astrojs/vercel` 11.x. It has no `astro preview`, so the CI smoke test would need a preview URL instead.
- **Render.** Native Node, Frankfurt region, CLI with `deploys create --wait` and JSON output. MCP server is **GA** (2025-08-21). Rollback has no CLI command and goes through the dashboard or the REST API. Pricing:
  - Free instances sleep after 15 min with roughly a 1 min cold start, so they're not usable.
  - A Starter instance is $7/month on top of the $0 Hobby workspace (plan model enforced 2026-08-01).
  - There's no CDN in front of static assets.
- **Netlify.** Strong docs and MCP. But function region is US (`cmh`) unless you're on Pro ($20/month), which puts every SSR request on a US↔EU round trip to Supabase. Other limits:
  - Hard 60 s function limit, risky for a synchronous import of a few hundred records.
  - Free credits pause every site when they run out, and each production deploy costs 15 credits.
  - No `rollback` CLI command.

  Weighted down on cost and region.
- **Fly.io.** Good CLI (`fly deploy`, `fly logs`, image-based rollback) and docs (`llms.txt`, MDX on GitHub). Downsides:
  - No free tier for new orgs.
  - You own the container and Dockerfile.
  - `waw` (Warsaw) region is deprecated, so you'd use `fra`/`ams`.
  - `fly mcp server` is **experimental**.
- **Railway.** Good docs (`llms.txt`). Weaknesses for this project:
  - Rollback is dashboard-only.
  - Exit codes aren't documented.
  - `railway.json` config-as-code is **deprecated**, with a hard cutoff 2026-12-01.
  - Remote MCP is in **preview** (2026-04-17).
  - The only EU region is Amsterdam.

### Shortlisted Platforms

#### 1. Cloudflare Workers (Recommended)

Zero migration. The developer knows it. Fully serverless (no container, OS or port to misconfigure), and the CLI covers every routine operation. Static assets are served from the edge, and SSR runs in a Warsaw-area PoP. Costs $0–5/month. The gaps are tooling churn (wrangler → cf, MCP server consolidation) and the tight Free-plan CPU and subrequest limits.

#### 2. Vercel

The most polished deploy API (`--prod`, `promote`, `rollback`, `inspect --wait`) and the best agent docs. Node runtime with a 300 s function limit and no CPU-time cliff, so the LLM import is less constrained. It ranks second because moving requires an adapter swap and a CI smoke-test rework, its MCP server is still beta, and the developer has no stated familiarity with it.

#### 3. Render

Plain long-running Node (no runtime quirks), Frankfurt region, GA MCP server, and deploys that report success or failure through exit codes. It ranks third because a usable always-on instance costs $7/month, it's not serverless (you size the instance), there's no CDN, and rollback isn't a CLI command.

## Anti-Bias Cross-Check: Cloudflare Workers

### Devil's Advocate — Weaknesses

1. **Free-plan CPU cliff.** 10 ms CPU per request covers React SSR, Supabase JWT handling and parsing several hundred CSV rows on a good day. On a bad day the request fails with error 1102 ("exceeded resource limits"), intermittently and only in production. In practice the MVP needs Workers Paid ($5/month).
2. **Subrequest budget.** Free allows 50 subrequests per request. A naive import that makes one OpenRouter call or one Supabase insert per record breaks at about 45 records. The code must batch: one or a few LLM calls per import, and bulk inserts.
3. **Tooling churn.** wrangler is being replaced by the `cf` CLI (open beta; one secondary source says it needs Node ≥ 22.18, while `.nvmrc` pins 22.14). The Observability, Builds and Bindings MCP servers are deprecated. Cloudflare's own Astro guide still shows the old `_worker.js` setup. Agents trained on older material will produce wrong commands such as `wrangler pages deploy` and `Astro.locals.runtime`.
4. **Thin observability on Free.** Workers Logs keeps 3 days on Free (7 on Paid), and logs pricing changes 2026-12-01. Preview-alias version URLs don't show up in `wrangler tail` or Workers Logs, so a broken preview is hard to debug.
5. **The workerd runtime isn't Node.** `nodejs_compat` polyfills the rest through unenv. Any future npm dependency that touches an unpolyfilled Node API (for example some CSV parsers or charting server code) throws `[unenv] … not implemented`, but only in the Worker build.

### Pre-Mortem — How This Could Fail

The team deployed on the Free plan because "it's only a few family members." The first real import, 400 rows from May, worked locally, because `astro dev` doesn't enforce production CPU limits. In production it failed half the time with error 1102. The import code had been written as one OpenRouter call per record, so it also hit the 50-subrequest cap, and records past #45 silently came back as `?`. A late-night fix moved prediction to a single batched prompt, which worked. Then the developer asked the agent to "add a preview environment." The agent ran `wrangler versions upload --preview-alias`, and the preview shared production secrets. A test import from a PR landed in a real family member's account in production Supabase. Meanwhile the agent's notes still referenced `wrangler pages deploy`, and a minor bump of `@astrojs/cloudflare` (14.4 requires `astro@^7.4`) broke a CI build nobody looked at for a week. The logs that might have explained it had expired after 3 days. Six months in, nobody trusted the deploy pipeline or the data. The root cause wasn't Cloudflare itself but untested assumptions about its limits.

### Unknown Unknowns

- **Pages is no longer an option.** `@astrojs/cloudflare` v14 supports Workers only. `tech-stack.md` still says `deployment_target: cloudflare-pages`, which is stale. The deploy command is `wrangler deploy`, never `wrangler pages deploy`.
- **Previews share production secrets.** Worker versions (including `--preview-alias` URLs) use the same bindings and secrets as production, so a preview talks to production Supabase and OpenRouter. True isolation needs a separate Worker (for example `fin-assist-preview`) with its own secrets and its own Supabase project.
- **Hidden KV provisioning.** The adapter configures a `SESSION` KV binding that is auto-provisioned on deploy. A scoped API token without KV permission fails the deploy. Alternatively, set `session: false` if Astro sessions aren't used (auth uses Supabase cookies).
- **Worker name.** `wrangler.jsonc` still says `"name": "10x-astro-starter"`. The first deploy would create a Worker and URL with that name, and renaming later creates a second Worker rather than moving the first.
- **Supabase region matters more than the Worker's.** The Worker runs near the user (Warsaw/Frankfurt PoP). If the Supabase project lives in `us-east-1`, every SSR query crosses the Atlantic. Create the Supabase project in `eu-central-1` (Frankfurt).

## Operational Story

- **Preview deploys**: `npx wrangler versions upload --preview-alias pr-<n>` produces `pr-<n>-fin-assist.<subdomain>.workers.dev` without promoting it to production. These previews **share production secrets**, so for the MVP treat them as production-equivalent. Don't run test imports on them. Add a separate `fin-assist-preview` Worker with its own Supabase project before relying on previews. Preview URLs are public; app auth protects the data, and Cloudflare Access can gate the hostname if needed. Worker Previews (`wrangler preview`, wrangler ≥ 4.135) are reportedly **open beta** (unconfirmed in official docs, checked 2026-10-23), so don't depend on them.
- **Secrets**:
  - **Production:** `SUPABASE_URL`, `SUPABASE_KEY` and later `OPENROUTER_API_KEY` live as Workers Secrets (`npx wrangler secret put <NAME>`). They're readable only by the Worker at runtime and listed by name with `wrangler secret list`.
  - **Local:** `.dev.vars` (gitignored).
  - **CI:** `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` as GitHub Secrets.
  - **Rotation:** issue a new key in Supabase/OpenRouter → `wrangler secret put` (this creates a new deployment immediately) → revoke the old key.
- **Rollback**: `npx wrangler deployments list` → `npx wrangler rollback <version-id> --message "reason"`. Takes effect in seconds globally. Supabase migrations are **not** rolled back. Write migrations to be backward-compatible with the previous Worker version, and fix forward for schema changes.
- **Approval**:
  - **Agent may run unattended:** `astro build`, `astro preview`, `wrangler versions upload` (non-promoted), `wrangler deployments list`, `wrangler tail`, `wrangler secret list`.
  - **Human runs or approves:** `wrangler deploy` to production (or the merge to `master` once CI auto-deploys), `wrangler secret put` on production, `wrangler rollback`.
  - **Human only, in the dashboard:** deleting the Worker or a KV namespace, Supabase production migrations that drop data, and rotating primary keys.
- **Logs**:
  - **Runtime:** `npx wrangler tail fin-assist --format json` for a live stream (production deployment only).
  - **Historical:** Workers Logs (`observability.enabled: true` is already set in `wrangler.jsonc`), via the dashboard or the Cloudflare API MCP server.
  - **Build/CI:** `gh run list` / `gh run view <id> --log`.

## Risk Register

| Risk | Source | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| Free-plan 10 ms CPU limit kills imports (error 1102) | Devil's advocate | H | H | Start on Workers Paid ($5/month), or measure CPU on a 500-row import in a preview before going live on Free |
| 50-subrequest cap breaks per-record LLM or DB calls | Devil's advocate / Pre-mortem | H | H | Batch predictions (one or a few OpenRouter calls per import) and use bulk Supabase inserts; add a lesson via `/10x-lesson` |
| Preview versions write to production Supabase | Unknown unknowns / Pre-mortem | M | H | Treat previews as prod until a separate `fin-assist-preview` Worker + Supabase project exists; no test imports on preview URLs |
| Stale guidance: `wrangler pages deploy`, `Astro.locals.runtime`, `_worker.js` | Unknown unknowns / Devil's advocate | H | M | Fix `tech-stack.md` hint to `cloudflare-workers`; record in CLAUDE.md: Workers only, `import { env } from "cloudflare:workers"` |
| Adapter caret upgrade breaks build (14.4 needs `astro@^7.4`) | Pre-mortem | M | M | Commit the lockfile, upgrade astro and adapter together, run `npm run smoke` after upgrades (already in CLAUDE.md) |
| Deploy fails with a scoped token due to auto-provisioned `SESSION` KV | Unknown unknowns | M | L | Grant Workers KV Storage:Edit on the token, or set `session: false` in the adapter config |
| Worker named `10x-astro-starter` in production | Unknown unknowns | H | L | Rename to `fin-assist` in `wrangler.jsonc` before the first deploy |
| Supabase project in a far region adds latency to every request | Unknown unknowns | M | M | Create the Supabase project in `eu-central-1` (Frankfurt) |
| Unpolyfilled Node API in a new dependency fails only in the Worker build | Devil's advocate | M | M | Prefer fetch-based, workerd-compatible libraries; CI's preview smoke test already runs in workerd |
| Logs expire (3 days Free) or pricing changes 2026-12-01 | Devil's advocate / Research finding | M | L | Re-check Workers Logs pricing in December; upgrade to Paid (7 days) if debugging needs it |
| wrangler → `cf` CLI migration (`cf` open beta; may need Node ≥ 22.18) | Research finding | L | M | Stay on wrangler 4.x for MVP; revisit when `cf` reaches GA (18-month overlap) |
| Deprecated Cloudflare MCP servers (Observability/Builds/Bindings; `/sse` retired 2026-10-22) | Research finding | L | L | Use only the Cloudflare API MCP (`mcp.cloudflare.com/mcp`) and the Docs MCP, if MCP is added at all |

## Getting Started

Verified against the pinned versions: astro 7.3.2, @astrojs/cloudflare 14.3.1, wrangler 4.131.1. `astro dev` and `astro preview` already run in workerd via the Cloudflare Vite plugin, so `wrangler dev` isn't needed.

1. **Rename the Worker.** In `wrangler.jsonc` set `"name": "fin-assist"`. In `context/foundation/tech-stack.md`, change `deployment_target: cloudflare-pages` to `cloudflare-workers`.
2. **Authenticate.** Run `npx wrangler login` locally. For CI/agent use, create a scoped API token with Workers Scripts:Edit and Workers KV Storage:Edit on this account only (no DNS, no billing), and export it as `CLOUDFLARE_API_TOKEN` plus `CLOUDFLARE_ACCOUNT_ID`. Check with `npx wrangler whoami`.
3. **Set production secrets.** Use an EU-region Supabase project: `npx wrangler secret put SUPABASE_URL`, then `npx wrangler secret put SUPABASE_KEY`. Add `OPENROUTER_API_KEY` when prediction lands, and declare it in `astro.config.mjs` `env.schema`.
4. **Build and deploy to Workers** (not Pages): `npm run build && npx wrangler deploy`. Note the `https://fin-assist.<subdomain>.workers.dev` URL.
5. **Verify.** Run `BASE_URL=https://fin-assist.<subdomain>.workers.dev npm run smoke`, then `npx wrangler tail fin-assist --format json` while signing in. Confirm a rollback target exists with `npx wrangler deployments list`.

## Out of Scope

The following were not evaluated in this research:
- Docker image configuration
- CI/CD pipeline setup (auto-deploy on merge to `master` via `cloudflare/wrangler-action@v4` or Workers Builds is a follow-up)
- Production-scale architecture (multi-region, HA, DR)
