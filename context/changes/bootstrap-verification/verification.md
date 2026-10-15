---
bootstrapped_at: 2026-10-15T19:49:36Z
starter_id: 10x-astro-starter
starter_name: "10x Astro Starter (Astro + Supabase + Cloudflare)"
project_name: fin-assist
language_family: js
package_manager: npm
cwd_strategy: git-clone
bootstrapper_confidence: first-class
phase_3_status: ok
audit_command: "npm audit --json"
---

## Hand-off

```yaml
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
```

### Why this stack

A solo, after-hours build of a private family expense categorizer with a 3-week MVP needs a starter that ships auth, a database, and deployment out of the box, so the time goes into CSV import, rules, and prediction rather than plumbing. The 10x Astro Starter (Astro + React + TypeScript + Tailwind + Supabase + Cloudflare) is the recommended default for a JavaScript/TypeScript web app and clears all four agent-friendly gates. Supabase covers email/password login with admin-created accounts and, through row-level security, the PRD's hard guardrail that no user's data is visible to anyone else, including the admin; those policies must land with the first table. Astro API routes handle CSV upload and the call to an outside prediction service via the already-installed Vercel AI SDK, with Zod validating both the bank CSV rows and prediction responses. Payments, realtime, and background jobs are out of scope per the PRD. Deployment targets Cloudflare, and CI runs on GitHub Actions with auto-deploy on merge to main.

## Pre-scaffold verification

| Signal      | Value                                                        | Severity | Notes                                                        |
| ----------- | ------------------------------------------------------------ | -------- | ------------------------------------------------------------ |
| npm package | not run                                                      | —        | cmd_template starts with `git clone`; no `create-*` CLI to resolve |
| GitHub repo | przeprogramowani/10x-astro-starter last pushed 2026-09-12T21:16:08Z | fresh    | from card.docs_url (https://github.com/przeprogramowani/10x-astro-starter) |

## Scaffold log

**Resolved invocation**: `git clone https://github.com/przeprogramowani/10x-astro-starter .bootstrap-scaffold && cd .bootstrap-scaffold && npm install`
**Strategy**: git-clone (cloned `.bootstrap-scaffold/.git/` deleted before move-up)
**Exit code**: 0
**Files moved**: 48 files (excluding `node_modules/`) across 20 top-level entries — `AGENTS.md`, `astro.config.mjs`, `components.json`, `.env.example`, `eslint.config.js`, `.github/`, `.husky/`, `node_modules/`, `.nvmrc`, `package.json`, `package-lock.json`, `.prettierrc.json`, `public/`, `README.md`, `scripts/`, `src/`, `supabase/`, `tsconfig.json`, `.vscode/`, `wrangler.jsonc`
**Conflicts (.scaffold siblings)**: `CLAUDE.md.scaffold`
**context/ handling**: scaffold shipped no `context/` paths; nothing dropped
**.gitignore handling**: append-merged (2 new lines under `# from 10x-astro-starter`: `# cloudflare`, `.dev.vars`)
**.bootstrap-scaffold cleanup**: deleted

**npm install notes**: 658 packages added (node v24.21.0, npm 11.19.0). npm reported that install scripts for 3 packages were not run because they are not covered by `allowScripts`: `esbuild@0.28.2`, `esbuild@0.28.1`, `workerd@1.20260911.1` (all `postinstall: node install.js`). Review with `npm install-scripts ls` and approve with `npm install-scripts approve <pkg>` if dev/build commands fail. Note that the starter's `.nvmrc` pins a Node version; the local Node is v24.21.0.

## Post-scaffold audit

**Tool**: `npm audit --json` (exit code 1, informational)
**Summary**: 0 CRITICAL, 9 HIGH, 2 MODERATE, 0 LOW
**Direct vs transitive**: 0/1/0/0 direct of total 0/9/2/0 — the one direct finding is `wrangler` (devDependency), which is vulnerable only through its `miniflare` dependency. Every advisory reports `fixAvailable: true` (`npm audit fix`, no major-version bump flagged).

#### CRITICAL findings

None.

#### HIGH findings

- **wrangler** 4.131.1 (direct, devDependency): vulnerable range `4.16.0 - 4.148.0`, via `miniflare`. Fix available.
- **@cloudflare/vite-plugin** 1.54.8 (transitive): vulnerable range `1.2.3 - 1.63.0`, via `miniflare`, `wrangler`. Fix available.
- **miniflare** 5.20260911.0-alpha (transitive): vulnerable range `4.20250508.3 - 5.20261006.0-alpha`, via `sharp`, `undici`. Fix available.
- **undici** 7.29.0 (transitive): range `7.0.0 - 7.29.0`. Advisories: GHSA-3wwx-pv8p-q78v, GHSA-pmjh-fq2x-6v4x, GHSA-r53p-7pc4-xj5r, GHSA-rfgv-xxqx-mfg5, GHSA-3xpg-4rpp-hhhm, GHSA-2jfj-6hjv-fm6j, GHSA-2gqq-gqf2-x968, GHSA-w293-vg96-wgc3, GHSA-8436-99hf-9mmv, GHSA-rx4f-c7p8-82vq (DoS, response splitting, cross-user cookie disclosure, TLS validation bypass in BalancedPool, unsafe-method response caching). Fix available.
- **sharp** 0.35.4 (transitive): range `<0.35.5`. GHSA-wq5f-xc86-pv6w (librsvg CVE-2026-96889). Fix available.
- **devalue** 5.9.2 (transitive): range `<=5.9.2`. GHSA-j22f-vq7h-c4qm, GHSA-hx4r-w6wj-j8fg, GHSA-mcm9-63f2-9j32, GHSA-wf3x-273g-mvxv, GHSA-x5rw-q4pp-hg5g, GHSA-4q55-j62x-fr9h (serialization/CPU amplification, `__proto__` rejection bypass). Fix available.
- **http-cache-semantics** 4.2.0 (transitive): range `<=4.2.0`. GHSA-ch52-4w7c-c8xp (max-stale can disclose cross-user cached responses). Fix available.
- **source-map-js** 1.2.1 (transitive): range `1.0.0 - 1.2.1`. GHSA-68fv-2mgg-jv7q (event-loop DoS). Fix available.
- **brace-expansion** 1.1.18 / 5.0.9 (transitive): ranges `<=1.1.20 || 4.0.0 - 5.0.11`. GHSA-q2hr-2g5m-vwhr, GHSA-qhr7-859c-m2p7, GHSA-6j4f-fj2g-mc7p (CPU / stack-exhaustion DoS). Fix available.

#### MODERATE findings

- **fast-uri** 3.1.7 (transitive): range `3.0.0 - 3.1.7`. GHSA-hrr3-gc8f-f4qj (inconsistent host case normalization). Fix available.
- **smol-toml** 1.8.0 (transitive): range `<=1.8.0`. GHSA-r4xh-jqrq-34v2 (quadratic-time parse). Fix available.

#### LOW / INFO findings

None.

## Hints recorded but not acted on

| Hint                    | Value                |
| ----------------------- | -------------------- |
| bootstrapper_confidence | first-class          |
| quality_override        | false                |
| path_taken              | standard             |
| self_check_answers      | null                 |
| team_size               | solo                 |
| deployment_target       | cloudflare-pages     |
| ci_provider             | github-actions       |
| ci_default_flow         | auto-deploy-on-merge |
| has_auth                | true                 |
| has_payments            | false                |
| has_realtime            | false                |
| has_ai                  | true                 |
| has_background_jobs     | false                |

## Next steps

Next: a future skill will set up agent context (CLAUDE.md, AGENTS.md). For now, your project is scaffolded and verified — happy hacking.

Useful manual steps in the meantime:
- `git init` (if you have not already) to start your own repo history.
- Review any `.scaffold` siblings the conflict policy created and decide which version of each file to keep.
- Address audit findings per your project's risk tolerance — the full breakdown is in this log.
