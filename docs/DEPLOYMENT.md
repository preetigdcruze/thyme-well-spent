# CI/CD

## What runs
| Workflow | Trigger | What it does |
|---|---|---|
| `ci.yml` → **verify** | every PR, push to `main` | `tsc`, apply migrations to an empty Postgres, **fail on schema/migration drift**, seed, unit + integration tests, production build |
| `ci.yml` → **e2e** | after verify | starts the built app and runs `scripts/qa.mjs` (signup → onboarding → plan → swap → log → grocery/pantry → search → isolation); screenshots are uploaded as an artifact |
| `deploy.yml` → **preview** | every PR from this repo | Vercel preview deployment, URL commented on the PR |
| `deploy.yml` → **production** | push/merge to `main` | `prisma migrate deploy` on the production DB, then Vercel production deploy |
| Dependabot | weekly (npm), monthly (actions) | grouped minor/patch update PRs |

Deploy jobs skip themselves (with a notice) until the secrets below exist, so CI is green from day one.

## One-time setup
1. **Database** – create a Postgres on [Neon](https://neon.tech) (or Supabase). Make two branches/databases: *production* and *preview*. Use the pooled connection string, e.g. `postgresql://user:pass@host/db?sslmode=require`.
2. **Vercel** – import the GitHub repo as a project (Framework: Next.js). Then set **Environment Variables** in Vercel:
   | Variable | Production | Preview |
   |---|---|---|
   | `DATABASE_URL` | production DB | preview DB |
   | `AUTH_SECRET` | `openssl rand -base64 32` | different value |
   | `AUTH_URL` | your production URL | *(omit; `AUTH_TRUST_HOST=true`)* |
   | `AUTH_TRUST_HOST` | `true` | `true` |
   | `AI_PROVIDER` | `mock`, `openai` or `anthropic` (+ matching key) | `mock` |
   | `NEXT_PUBLIC_DEMO` | `off` | *(optional)* |
   Optional: `AUTH_GOOGLE_ID/SECRET`, `AUTH_APPLE_ID/SECRET`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`.
3. **GitHub → Settings → Secrets and variables → Actions**, add:
   - `VERCEL_TOKEN` – from vercel.com/account/tokens
   - `VERCEL_ORG_ID` and `VERCEL_PROJECT_ID` – from `.vercel/project.json` after running `npx vercel link` locally
   - `PRODUCTION_DATABASE_URL` – same production URL as in Vercel (used to run migrations)
4. **GitHub → Settings → Environments**: create `production`; optionally require a reviewer to gate production deploys.
5. **Branch protection** on `main`: require the `Typecheck, test, build` and `Browser QA walkthrough` checks.
6. Disable Vercel's own Git auto-deploys (Project → Git → *Ignored Build Step* = `exit 0`) so deployments only come from this pipeline.

## Notes
- Production is **not seeded** with the demo account; run `npm run db:seed` manually against a staging DB only if you want sample data. The seed also upserts the recipe catalogue, which production needs: run it once with `DATABASE_URL=<prod> npm run db:seed` and then delete `demo@thyme.app` if you don't want it.
- Migrations: create with `npm run db:migrate` locally and commit the generated folder. CI blocks PRs whose `schema.prisma` has changes without a migration.
- Rollback: redeploy a previous build from the Vercel dashboard (migrations are forward-only; write backwards-compatible migrations).
