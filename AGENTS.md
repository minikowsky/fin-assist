# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

fin-assist is an early-stage Astro 7 app (still close to the starter template: one page, `Welcome.astro`, `Layout.astro`) deployed to Cloudflare Workers. The stack is wired up but not yet used in app code:

- **Astro + React** (`@astrojs/react`, `jsx: react-jsx`) for pages and interactive islands
- **Tailwind CSS v4** via the `@tailwindcss/vite` plugin (configured in `astro.config.mjs`, styles in `src/styles/global.css`, no `tailwind.config`)
- **Cloudflare adapter** (`@astrojs/cloudflare`); `wrangler.jsonc` points `main` at the adapter's server entrypoint and serves `./dist` as `ASSETS`
- **Supabase** (`@supabase/supabase-js`, local config in `supabase/`)
- **Vercel AI SDK** (`ai`, `@ai-sdk/anthropic`, `@ai-sdk/google`) for LLM calls
- **Zod 4** for validation

## Commands

Requires Node >= 22.12.0. No test runner or linter is configured.

- `npm run dev`: dev server at `localhost:4321`
- `npm run build`: build to `./dist/`
- `npm run preview`: preview the build locally
- `npm run generate-types`: `wrangler types` (regenerates Cloudflare env typings; `tsconfig.json` includes `worker-configuration.d.ts`)

When starting the dev server, use background mode:

```
astro dev --background
```

Manage the background server with `astro dev stop`, `astro dev status`, and `astro dev logs`.

## Notes

- Secrets live in `.env` (gitignored). On Cloudflare, runtime bindings/env come from the Worker environment, not `process.env`; keep that in mind when reading Supabase/AI provider keys in server code.
- `CLAUDE.md` is a symlink to `AGENTS.md`; edit `AGENTS.md`.

## Documentation

Full documentation: https://docs.astro.build

Consult these guides before working on related tasks:

- [Adding pages, dynamic routes, or middleware](https://docs.astro.build/en/guides/routing/)
- [Working with Astro components](https://docs.astro.build/en/basics/astro-components/)
- [Using React, Vue, Svelte, or other framework components](https://docs.astro.build/en/guides/framework-components/)
- [Adding or managing content](https://docs.astro.build/en/guides/content-collections/)
- [Adding styles or using Tailwind](https://docs.astro.build/en/guides/styling/)
- [Supporting multiple languages](https://docs.astro.build/en/guides/internationalization/)
