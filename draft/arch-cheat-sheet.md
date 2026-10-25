# fin-assist: architecture cheat sheet

How the pieces of the stack fit together and what each one is responsible for.
Based on the repo as of 2026-10-25 (`package.json`, `astro.config.mjs`, `wrangler.jsonc`, `src/`, `.github/workflows/ci.yml`).

## The big picture

```
                        DEVELOPMENT (your laptop)                              PRODUCTION
 ┌───────────────────────────────────────────────────────────┐   ┌─────────────────────────────────────┐
 │ Node.js 22 ── runs the tooling: npm, astro CLI, wrangler, │   │ Cloudflare Workers (edge, workerd)  │
 │               eslint, prettier, tsc, supabase CLI         │   │  ├─ Worker: Astro SSR bundle        │
 │                                                           │   │  │   middleware → pages / API       │
 │ TypeScript + Astro + React + Tailwind  (source in src/)   │   │  └─ Static Assets: dist/ (CSS,      │
 │        │  npm run build  (Vite)                           │   │      JS islands, images)            │
 │        ▼                                                  │   └───────────────┬─────────────────────┘
 │ dist/  ── Worker bundle + static assets                   │                   │ HTTPS (supabase-js)
 │        │  npx wrangler deploy  ─────────────────────────────────────►         ▼
 │                                                           │   ┌──────────────────────────────────┐
 │ Docker ── runs local Supabase (npx supabase start)        │   │ Supabase Cloud                   │
 │           Postgres :54322, API :54321, Studio :54323      │   │  Auth (users, sessions)          │
 └───────────────────────────────────────────────────────────┘   │  Postgres + RLS                  │
                                                                 └──────────────────────────────────┘
```

Request path in production:

```
Browser ──► Cloudflare edge ──► static file? ──yes──► served from Static Assets (no Worker CPU)
                                     │ no
                                     ▼
                          Worker (Astro SSR, workerd runtime)
                            1. src/middleware.ts: reads Supabase cookies,
                               calls supabase.auth.getUser(), sets locals.user,
                               redirects guests away from /dashboard
                            2. page (.astro) renders HTML, or API route (POST) runs
                            3. talks to Supabase over HTTPS when needed
                          ◄── HTML + Set-Cookie
Browser loads the React island JS (client:load) and hydrates the interactive parts
```

## Responsibilities, one by one

| Tech | Role in this project | Runs where | Key files |
|---|---|---|---|
| **TypeScript** | Language for all code. Types catch errors before runtime; `astro check` and type-aware ESLint enforce them in CI. | Compile time only (stripped by Vite) | `tsconfig.json` (`@/*` → `src/*`), `src/env.d.ts`, `src/types.ts` |
| **Astro 7** | The web framework. Routes files in `src/pages/` to URLs, renders pages on the server (`output: "server"`), runs middleware, hosts API routes, bundles everything with Vite. | Server (in the Worker); at build time on Node | `astro.config.mjs`, `src/pages/`, `src/layouts/`, `src/middleware.ts` |
| **React 19** | Interactive pieces only ("islands"), e.g. the sign-in and sign-up forms. Astro renders them to HTML on the server, then ships JS so they hydrate in the browser (`client:load`). | Server (first render) + browser | `src/components/auth/*.tsx`, `src/components/ui/` (shadcn/ui) |
| **Tailwind 4** | Styling with utility classes, compiled into one CSS file at build time via the Vite plugin. `cn()` merges class names; shadcn/ui components are built on it. | Build time → static CSS | `src/styles/global.css`, `src/lib/utils.ts`, `components.json` |
| **Supabase** | Backend as a service: **Auth** (email/password, sessions in cookies) and **Postgres** (app data, protected by Row-Level Security). The app is only a client of it; Supabase holds the data. | Supabase Cloud (prod), Docker (local) | `src/lib/supabase.ts`, `src/pages/api/auth/*.ts`, `supabase/config.toml`, `supabase/migrations/` |
| **Cloudflare (Workers)** | Production hosting. Runs the Astro server bundle in the `workerd` runtime at the edge and serves `dist/` as static assets. Stores secrets (`SUPABASE_URL`, `SUPABASE_KEY`), provides logs. | Cloudflare edge | `wrangler.jsonc`, `@astrojs/cloudflare` adapter |
| **wrangler** | Cloudflare's CLI. Deploys (`wrangler deploy`), manages secrets (`wrangler secret put`), streams logs (`wrangler tail`), lists versions and rolls back. Also the engine behind the local workerd runtime. | Your machine / CI (on Node) | `wrangler.jsonc`, `.dev.vars`, generated `.wrangler/deploy/config.json` |
| **Node.js 22** | Runs **the tooling**, not production: npm, `astro dev/build/preview`, wrangler, supabase CLI, ESLint, Prettier, `scripts/smoke.mjs`. | Your machine + GitHub Actions | `.nvmrc`, `package.json` |
| **Docker** | Runs a full local Supabase stack (Postgres, Auth, Studio, mail catcher…) as containers so you can develop and test without touching the cloud project. Also used in CI for the smoke test. Not used for deploying the app. | Your machine + CI | `supabase/config.toml` |

## How they connect (the important seams)

### Astro ↔ Cloudflare: the adapter
- `@astrojs/cloudflare` turns the Astro build into a **Worker script + static assets** in `dist/`.
- `wrangler.jsonc` points `main` at the adapter's server entrypoint and `assets.directory` at `./dist`.
- `nodejs_compat` flag: workerd is **not Node.js**. It is a V8 runtime with Web APIs (`fetch`, `Request`, `Response`) plus a subset of Node APIs. Code that needs `fs`, native modules or long-lived TCP connections won't work there.
- Deliberately turned off to avoid extra Cloudflare bindings: Astro sessions (would need KV) and image transforms (`imageService: "passthrough"`, would need Cloudflare Images).

### Astro ↔ Node vs workerd: same code, two runtimes
- **Node** runs the build tools and the CLI processes.
- **workerd** runs your app code: `npm run dev` and `npm run preview` already execute it in workerd (through the Cloudflare Vite plugin / wrangler), so local behaviour matches production.
- So: Node = the workshop, workerd = the factory floor.

### Astro ↔ Supabase: cookie-based auth
1. The user submits the React `SignInForm` → `POST /api/auth/signin` (Astro API route).
2. The route creates a server client (`@supabase/ssr`) and calls `signInWithPassword`. Supabase returns a session that gets written to **cookies** through Astro's `cookies.set`.
3. On every later request `src/middleware.ts` builds the client from the cookies, calls `auth.getUser()` (verified against Supabase) and puts the user on `context.locals.user`.
4. Pages read `Astro.locals.user`; `PROTECTED_ROUTES` redirect guests to `/auth/signin`.
- Secrets come from `astro:env/server` (declared in `astro.config.mjs`). They are server-only, never shipped to the browser.
- Data safety lives **in the database**: every table gets RLS policies, so even a buggy query can only see the signed-in user's rows.

### Astro ↔ React ↔ Tailwind: rendering
- `.astro` files = static layout and content, zero JS by default.
- `.tsx` files = interactive widgets, hydrated only where a `client:*` directive is used.
- Tailwind classes work in both; the build produces a single CSS file served as a static asset.

### Supabase ↔ Docker: local backend
- `npx supabase start` uses Docker to start Postgres (`:54322`), the API gateway/Auth (`:54321`), Studio (`:54323`) and the mail catcher (`:54324`).
- Point `.env` / `.dev.vars` at the local `API_URL` + anon key to develop offline; point at the cloud project for production.
- Migrations in `supabase/migrations/` are applied locally (`supabase db reset`) and pushed to the cloud (`supabase db push`).

### wrangler ↔ Cloudflare: shipping
- `npm run build` → `dist/` → `npx wrangler deploy` uploads exactly what is in `dist/`. Always build right before deploying.
- Workers only. `wrangler pages deploy` is the wrong command for this project.
- Production secrets: `npx wrangler secret put SUPABASE_URL` (and `SUPABASE_KEY`). Local equivalents live in `.dev.vars` (gitignored).

## Config & secrets map

| Where | Holds | Used by |
|---|---|---|
| `.env` | `SUPABASE_URL`, `SUPABASE_KEY` | Astro build / Node tooling |
| `.dev.vars` | the same keys | wrangler / workerd locally (`dev`, `preview`) |
| Cloudflare Worker secrets | the same keys (prod values) | the deployed Worker |
| GitHub repo secrets | the same keys | CI build step |
| `astro.config.mjs` `env.schema` | declares the keys as server-only secrets | `astro:env/server` imports |

## CI (GitHub Actions)

- **ci job**: `npm ci` → `astro sync` → `lint` → `astro check` (TypeScript) → `build`.
- **smoke job**: starts a trimmed local Supabase in Docker → builds → runs `npm run preview` (workerd) → `npm run smoke` hits the auth flow.
- Deploy is currently manual with wrangler; auto-deploy on merge is the planned next step.

## Lifecycle in one line each

| Step | Command | Tech involved |
|---|---|---|
| Start local DB | `npx supabase start` | Docker, Supabase CLI |
| Develop | `npm run dev` | Node (Vite, Astro) + workerd |
| Check | `npm run lint`, `npx astro check` | Node, TypeScript, ESLint |
| Build | `npm run build` | Node, Vite, Astro, Tailwind, adapter |
| Preview prod build | `npm run preview` | wrangler / workerd |
| Deploy | `npm run build && npx wrangler deploy` | wrangler → Cloudflare |
| Logs | `npx wrangler tail` | wrangler, Cloudflare observability |
| Roll back | `npx wrangler rollback` | wrangler |

## Gotchas worth remembering

- **workerd ≠ Node.** Check that any new npm package works on Workers (Web APIs, no `fs`, no native addons).
- **Free plan: 10 ms CPU per request.** Waiting on `fetch` (Supabase, LLM calls) doesn't count; heavy CSV parsing might.
- **Never ship the service-role key.** `SUPABASE_KEY` is the anon/publishable key; RLS does the protecting.
- **Enable RLS on every new table**, with per-operation policies, in the same migration that creates it.
- **Docker group = root access** on your machine; rootless Docker is the stricter option.
