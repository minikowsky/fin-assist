---
starter_id: 10x-astro-starter
package_manager: npm
project_name: fin-assist
hints:
  language_family: js
  team_size: solo
  deployment_target: cloudflare-pages
  ci_provider: github-actions
  ci_default_flow: auto-deploy-on-merge
  bootstrapper_confidence: first-class
  path_taken: standard
  quality_override: false
  self_check_answers: null
  has_auth: true
  has_payments: false
  has_realtime: false
  has_ai: true
  has_background_jobs: false
---

## Why this stack

A solo, after-hours build of a private family expense categorizer with a 3-week MVP needs a starter that ships auth, a database, and deployment out of the box, so the time goes into CSV import, rules, and prediction rather than plumbing. The 10x Astro Starter (Astro + React + TypeScript + Tailwind + Supabase + Cloudflare) is the recommended default for a JavaScript/TypeScript web app and clears all four agent-friendly gates. Supabase covers email/password login with admin-created accounts and, through row-level security, the PRD's hard guardrail that no user's data is visible to anyone else, including the admin; those policies must land with the first table. Astro API routes handle CSV upload and the call to an outside prediction service via the already-installed Vercel AI SDK, with Zod validating both the bank CSV rows and prediction responses. Payments, realtime, and background jobs are out of scope per the PRD. Deployment targets Cloudflare, and CI runs on GitHub Actions with auto-deploy on merge to main.
