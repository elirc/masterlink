# Tooling and Build System

## Mental model

A monorepo has three compilation stories to keep straight: (1) how each app *builds for production*, (2) how each *runs in dev*, (3) how *shared packages* get consumed by both. Confusing them is the root of most "works on my machine" monorepo pain.

## This repo's answers

| Consumer | Dev | Prod | Shared packages consumed via |
| --- | --- | --- | --- |
| web | `next dev` | `next build` → `next start` | Next transpiles workspace deps (source-level) |
| worker | `tsx watch index.ts` | **also `tsx`** (`start: tsx index.ts`) — no build step; TS executed directly in prod | same source files, no bundling |
| mobile | Expo/Metro | EAS builds | Metro resolves workspace source |

Consequences worth articulating: the worker has **no compile gate in its run path** — type errors surface at require-time or never; hence its lone script `typecheck: tsc --noEmit` matters and isn't in CI ([critique risk 1](../03-architecture-and-patterns/06-architecture-critique.md)). The web app *does* get type-checked as part of `next build` (which CI runs).

## Yarn 4 specifics

- Pinned via `packageManager` + corepack (never global-install yarn 4).
- `resolutions` (root package.json) force-dedupes `react: 19.1.0` and RN-native deps — the duplicate-React guard ([08/01 Q8](../08-interview-prep/01-js-ts-node-deep-dive.md)).
- `.yarnrc.yml` sets `nodeLinker` behavior (check it — 51 bytes, likely `node-modules` linker for RN compat).
- `patch-package` in `postinstall` with `patches/` dir — dependency source edits that survive installs; read one patch to see *why* it exists before ever upgrading that dep (transferable habit).

## Prisma workflow

`prisma:generate` (client codegen — run after every schema pull/change; CI does — `playwright-tests.yml`), `prisma:dev` (create+apply migration in dev), `prisma:deploy` (apply committed migrations, prod/CI), `prisma:studio` (GUI). Consumers import `@linkwarden/prisma`, whose `index.ts` exports a **singleton** `PrismaClient` (cached on `global` to survive Next dev hot-reload — a classic Next+Prisma trick) with query logging switchable via `DEBUG=true`. That one-file facade means connection/logging policy lives in exactly one place.

## Docker story

[Dockerfile](../../../Dockerfile) (~2KB, multi-stage — read it once) + [docker-compose.yml](../../../docker-compose.yml) (app + Postgres; Meili optional). The e2e-relevant detail: CI builds the web app and runs `prisma:deploy` before boot — the canonical deploy order (migrate, then start). `release-container.yml` publishes images.

## Sharp edges checklist

☐ `yarn install` triggers Playwright browser download (web postinstall) — slow CI without caching (CI caches it — `playwright-tests.yml` cache steps) ☐ two lockfile-adjacent pin mechanisms (resolutions, patches) to check before upgrades ☐ `NEXT_PUBLIC_*` baked at build — image rebuild required to change ☐ vitest alias `@` → `apps/web` (`vitest.config.mts`) mirrors the web tsconfig path — tests break confusingly if they drift.

## Drills

1. `cat .yarnrc.yml patches/*` — write one sentence each: what's pinned and your best guess why.
2. Trace `import { prisma } from "@linkwarden/prisma"` from `postLink.ts` to the generated client file on disk.
3. From the Dockerfile, list the build stages and what would break if `prisma:generate` were skipped.

## Interview angle

[08/01 Q8](../08-interview-prep/01-js-ts-node-deep-dive.md) (monorepo resolution), plus two evergreen questions this file arms you for: "walk me through your deploy pipeline" (migrate→start ordering, image-baked env) and "how do you upgrade a risky dependency in a monorepo?" (resolutions/patches audit → typecheck matrix → per-app canary).
