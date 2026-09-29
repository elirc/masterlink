# Command Cheatsheet

All from repo root unless noted. **Status**: __verified__ = executed during curriculum authoring; __inferred__ = read from package.json scripts / CI config but not executed (no dependencies were installed in the authoring environment — see [verification-log.md](verification-log.md)).

| Task | Command | Status |
| --- | --- | --- |
| Enable toolchain | `corepack enable && corepack prepare yarn@4.12.0 --activate` | inferred (CI does exactly this) |
| Install | `yarn install` (runs playwright chromium download + patch-package postinstall) | inferred |
| Dev: web only | `yarn web:dev` | inferred |
| Dev: worker only | `yarn worker:dev` | inferred |
| Dev: both | `yarn concurrently:dev` | inferred |
| Prod build/start web | `yarn web:build` then `yarn web:start` | inferred |
| Start worker (prod) | `yarn worker:start` (tsx, no build step) | inferred |
| Prisma client codegen | `yarn prisma:generate` (after any schema change/pull) | inferred |
| New migration (dev) | `yarn prisma:dev` | inferred |
| Apply migrations (prod/CI) | `yarn prisma:deploy` | inferred |
| DB GUI | `yarn prisma:studio` | inferred |
| Unit tests | `yarn test` · single: `yarn test <name-fragment>` · CI-style: `yarn vitest run` | inferred |
| Coverage | `yarn coverage` | inferred |
| E2E | `yarn workspace @linkwarden/web e2e` · tagged: `yarn workspace @linkwarden/web playwright test --grep @login` | inferred (CI runs the grep form) |
| Lint (web only) | `yarn workspace @linkwarden/web lint` | inferred |
| Typecheck worker | `yarn workspace @linkwarden/worker typecheck` | inferred |
| Format all | `yarn format` | inferred |
| Docker stack | `docker compose up` (app + Postgres per docker-compose.yml) | inferred |
| Prisma query logging | set `DEBUG=true` (see `packages/prisma/index.ts`) | verified (code read) |
| Queue depth (SQL) | `SELECT count(*) FROM "Link" WHERE "lastPreserved" IS NULL;` | verified (schema read) |
| File inventory | `rg --files` / `git status --short` | verified |

No codegen beyond Prisma; no Makefile/justfile; migrations live in `packages/prisma/migrations/` (93 as of authoring).

Environment reminders: `.env` at repo root (scripts wrap with `dotenv --`); worker and web read the same file; `NEXT_PUBLIC_*` requires web **rebuild** to change.
