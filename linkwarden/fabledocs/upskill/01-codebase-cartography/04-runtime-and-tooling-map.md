# Runtime and Tooling Map

## Package manager & workspaces

Yarn **4.12.0** pinned via `packageManager` in [package.json](../../../package.json); activate with corepack (`.github/workflows/playwright-tests.yml:71-75`). Workspaces: `apps/*`, `packages/*`. Cross-package imports use `@linkwarden/*` names. Root `resolutions` pins `react: 19.1.0` across the tree (web + mobile must agree). `postinstall` runs the web app's Playwright install and `patch-package` (patches in `patches/`).

## Scripts you'll actually use (root package.json)

| Script | Does | Status |
| --- | --- | --- |
| `yarn web:dev` / `worker:dev` | dev servers (`next dev` / `tsx watch index.ts`), env loaded via `dotenv --` | __inferred__ |
| `yarn concurrently:dev` | both at once | __inferred__ |
| `yarn prisma:generate` / `prisma:dev` / `prisma:deploy` / `prisma:studio` | client codegen / create+apply dev migration / apply committed migrations / DB GUI | __inferred__ |
| `yarn test` / `yarn coverage` | vitest (unit; e2e excluded per [vitest.config.mts](../../../vitest.config.mts)) | __inferred__ |
| `yarn workspace @linkwarden/web e2e` | Playwright e2e | __inferred__ |
| `yarn workspace @linkwarden/web lint` | `next lint` — web only | __inferred__ |
| `yarn workspace @linkwarden/worker typecheck` | `tsc --noEmit` — worker only; **no repo-wide typecheck script exists** | __inferred__ |
| `yarn format` | prettier across workspaces | __inferred__ |

Full cheat sheet: [09-reference/command-cheatsheet.md](../09-reference/command-cheatsheet.md).

## Runtime boundaries

| Boundary | Runtime | Notes |
| --- | --- | --- |
| Browser | React 19, pages router | client components only (no RSC in pages router); i18n via next-i18next |
| Next server | Node — API routes + SSR | `getServerSideProps` wrapper in `apps/web/lib/client/getServerSideProps.ts`; API config overrides like `bodyParser: false` at `archives/[linkId].ts:17-22` |
| Worker | separate Node process via `tsx` (TS executed directly, no build) | supervised by `apps/worker/index.ts` |
| Headless browser | Playwright Chromium inside the worker | optionally remote via `PLAYWRIGHT_WS_URL` |
| Mobile | React Native (Expo) | consumes `packages/router` with Bearer auth |

Code in `packages/router` and `packages/lib` must run in **multiple** runtimes — that's why platform things (toast, Alert, session) are injected as parameters (`links.tsx:383-393`).

## Env vars, by concern (names from [.env.sample](../../../.env.sample); no secrets here)

- **Core:** `NEXTAUTH_URL`, `NEXTAUTH_SECRET`, `DATABASE_URL`
- **Behavior/limits:** `PAGINATION_TAKE_COUNT`, `MAX_LINKS_PER_USER`, `ARCHIVE_TAKE_COUNT`, `BROWSER_TIMEOUT`, `IMPORT_LIMIT`, various `*_MAX_BUFFER`
- **Feature switches (absence = off):** `MEILI_HOST`/`MEILI_MASTER_KEY` (search), `SPACES_*` (S3), `STRIPE_SECRET_KEY` (billing), `EMAIL_FROM`/`EMAIL_SERVER` (email), AI provider keys (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `OLLAMA_*`, …)
- **Security-relevant:** `ALLOW_PRIVATE_NETWORK_ACCESS` (disables SSRF guard — dangerous), `ALLOW_INSECURE_TLS`, `NEXT_PUBLIC_USER_CONTENT_DOMAIN` (XSS isolation), `NEXT_PUBLIC_DEMO`, `NEXT_PUBLIC_DISABLE_REGISTRATION`, `DISABLE_NEW_SSO_USERS`
- **Worker cadence:** `ARCHIVE_SCRIPT_INTERVAL`, `INDEX_TAKE_COUNT`, `NEXT_PUBLIC_RSS_POLLING_INTERVAL_MINUTES`

Transferable: `NEXT_PUBLIC_*` vars are **inlined into the client bundle at build time** — anything named that is world-readable and unchangeable without a rebuild. Spot the consequence: `NEXT_PUBLIC_DEMO` guards on the server (`links/index.ts:33-37`) are fine, but never put a secret behind `NEXT_PUBLIC_`.

## CI (what's enforced vs what isn't)

[playwright-tests.yml](../../../.github/workflows/playwright-tests.yml): Postgres 16 service → yarn install → `prisma:generate` → `web:build` → `prisma:deploy` → start web+worker → run Playwright **grep `@login` only**. Also: `release-container.yml` (Docker publish), `locale-action.yml` (crowdin), `check-branch.yml`.

Not enforced by CI: `yarn test` (vitest), lint, typecheck. *Possible risk* and a genuinely useful first contribution — see [06/01](../06-contribution-practice/01-good-first-tickets.md) Ticket 1.
