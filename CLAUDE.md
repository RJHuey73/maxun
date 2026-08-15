# CLAUDE.md

Guidance for Claude Code (claude.ai/code) working in this repository.

## What This Is

**Maxun** is an open-source, no-code web data platform: it turns websites into
structured APIs/spreadsheets via point-and-click "robots" (record a browsing
session and replay it), LLM-powered extraction, full-page scraping to
Markdown/HTML, site crawling, and automated search. This repo
(`RJHuey73/maxun`) is a fork of `getmaxun/maxun`.

It is a single npm package at the root (not an npm/yarn workspaces monorepo)
with a React frontend and an Express/TypeScript backend living side-by-side in
one `package.json`, plus a separately-published core library
(`maxun-core`) and a standalone browser microservice (`browser/`).

## Layout

| Path | Purpose |
|------|---------|
| `src/` | React (v18) + TypeScript frontend. `src/pages/` (Login, Register, MainPage, RecordingPage), `src/components/` (recorder, robot, run, browser, dashboard, integration, proxy, pickers, api, action, ui, icons), `src/api/` (axios clients per domain: auth, workflow, storage, proxy, integration, recording, webhook), `src/context/`, `src/routes/userRoute.tsx`, `src/shared/`, `src/helpers/`. Bundled by Vite; MUI + Emotion + styled-components for UI, `react-i18next` for i18n, socket.io-client for realtime run/recording updates. |
| `server/src/` | Express/TypeScript backend. `server.ts` (app wiring: CORS, sessions via `connect-pg-simple`, Socket.IO, route mounting, graceful shutdown), `index.ts` (barrel export of the public server surface), `task-runner.ts` (worker start/stop), `schedule-worker.ts` (cron-based robot scheduling), `mcp-worker.ts` (standalone MCP stdio server, excluded from the main server TS build), `browser-management/classes/` (`BrowserPool`, `RemoteBrowser` — the singleton pool driving Playwright sessions), `workflow-management/classes/` (`Interpreter`, `Generator`, `DocumentInterpreter` — replay/record a robot's recorded action workflow), `routes/` (`auth`, `record`, `workflow`, `storage`, `proxy`, `webhook`), `api/` (auto-mounted under `/api` — every file in this dir is `require`d and mounted at server start), `db/` (Sequelize: `models/` — `Robot`, `Run`, `User` — `migrations/`, `config/database.js`), `sdk/` (`browserAgent.ts`, `workflowEnricher.ts`, `selectorValidator.ts` for LLM-assisted/AI-mode extraction), `storage/` (Postgres + MinIO + Graphile Worker queue integration), `markdownify/`, `swagger/` (OpenAPI spec served at `/api-docs`). |
| `maxun-core/` | Separately-versioned, separately-published npm package (`maxun-core`, currently pinned `0.0.37` in the root/backend/frontend `package.json`s) that does the actual browser-driven data extraction: `src/interpret.ts` (workflow interpreter), `src/preprocessor.ts`, `src/browserSide/scraper.js` (injected into the page), `src/types/` (`workflow.ts`, `logic.ts`, `formats.ts`), `src/utils/concurrency.ts`. Has its own `package.json`/`tsconfig.json`/build. |
| `browser/` | Standalone browser microservice: exposes Playwright browsers over WebSocket (CDP) with `playwright-extra` + `puppeteer-extra-plugin-stealth` for anti-detection. Own `package.json`, own `Dockerfile`, runs as the `browser` Docker Compose service; backend connects to it via `BROWSER_WS_HOST`/`BROWSER_WS_PORT`. |
| `legacy/` | Older, superseded implementations kept for reference (`legacy/src/*.tsx` — old recorder/canvas/panel components; `legacy/server/*.ts` — old `pgboss`-based worker). Not part of the active build graph — don't extend these; find the current equivalent under `src/` / `server/src/` instead. |
| `docs/` | `self-hosting-docker.md`, `nginx.conf` — deployment docs (the bulk of user-facing docs live externally at docs.maxun.dev, not in this repo). |
| `public/` | Static frontend assets served by Vite/nginx. |
| Root config | `docker-compose.yml` (postgres, minio, backend, frontend, browser services), `Dockerfile.backend`, `Dockerfile.frontend`, `nginx.conf`, `ENVEXAMPLE` (canonical env var reference), `.sequelizerc` (points Sequelize CLI at `server/src/db/*`), `typedoc.json`. |

`package.backend.json` / `package.frontend.json` mirror the root
`package.json`'s deps split by concern (backend vs. frontend) with **pinned**
(non-`^`) versions — used to build the separate `Dockerfile.backend` /
`Dockerfile.frontend` images without pulling in the other half's
dependencies. When adding a dependency, add it to the root `package.json`
*and* the matching backend/frontend split file, keeping versions in sync.

## Commands

There is no top-level `test` script and no test suite in this repo (root,
`server/`, or `maxun-core/` — `maxun-core/package.json` declares
`"test": "jest"` but ships no Jest config or `*.test.*` files, and Jest isn't
a dependency). Validate changes by running the app and exercising the
affected flow manually, or via `/api-docs` for backend endpoints.

```bash
# Install (root deps cover both frontend and backend; maxun-core is separate)
npm install
cd maxun-core && npm install && cd ..
npx playwright install --with-deps chromium

# Run frontend + backend together (dev, with hot reload on the server)
npm run start:dev          # nodemon server + vite client, concurrently
npm run server:dev         # backend only (nodemon, ts-node via nodemon config)
npm run client             # frontend only (vite dev server)

# Run frontend + backend together (prod-style: build server first)
npm run start              # tsc build of server, then node server + vite client

# Build
npm run build               # frontend: vite build -> build/
npm run build:server        # backend: tsc -p server/tsconfig.json -> server/dist/
npm run mcp:build            # MCP stdio worker: tsc --project server/tsconfig.mcp.json

# Lint (frontend/root ESLint config only; no separate backend/maxun-core lint wired at root)
npm run lint

# Database (Sequelize CLI, config resolved via .sequelizerc)
npm run migrate
npm run migrate:undo
npm run migrate:undo:all
npm run migration:generate -- <name>
npm run seed
npm run seed:undo:all

# maxun-core (separate package)
cd maxun-core
npm run build    # clean + tsc -> build/
npm run lint      # eslint .
npm test          # declared but non-functional — no jest config/tests present
```

Local (non-Docker) setup needs Node.js, PostgreSQL, MinIO, and Redis running
and reachable per `ENVEXAMPLE`/`SETUP.md`. Frontend defaults to
`http://localhost:5173`, backend to `http://localhost:8080`. Docker Compose
(`docker-compose.yml`) additionally runs a dedicated `browser` service
(Playwright-over-WebSocket, ports `3001`/`3002`) that the backend's
`BrowserPool` connects to rather than launching Chromium in-process when
containerized.

## Architecture

- **Recording → workflow → replay.** A user's browser actions are captured
  client-side (`src/components/recorder/`, injected via
  `maxun-core/src/browserSide/scraper.js`) into a serialized workflow
  (`server/src/workflow-management/classes/Generator.ts`), persisted as a
  `Robot` (Sequelize model), and later replayed headlessly by
  `Interpreter`/`DocumentInterpreter` driving a pooled Playwright browser
  (`BrowserPool`/`RemoteBrowser`) to produce a `Run`.
- **`maxun-core` is the extraction engine, versioned independently.** It's
  consumed as a normal npm dependency (pinned in all three `package.json`s),
  not path-linked — bumping its behavior means publishing a new
  `maxun-core` version and updating the dependency, not just editing
  `maxun-core/src` and expecting the server to pick it up without a rebuild.
- **Two ways to drive a browser.** Locally/without Docker, the backend can
  launch Playwright directly (`playwright-core`). In the full Docker Compose
  stack, browser sessions are instead brokered over WebSocket/CDP by the
  separate `browser/` service (stealth-patched via `playwright-extra` +
  `puppeteer-extra-plugin-stealth`), configured via `BROWSER_WS_HOST` /
  `BROWSER_WS_PORT` / `BROWSER_HEALTH_PORT`.
- **Realtime updates via Socket.IO.** The single global `io` instance
  (exported from `server/src/server.ts`) pushes recording/run state to the
  client; the `/queued-run` namespace additionally tracks per-user queued-run
  recovery notifications (rooms keyed `user-${userId}`).
- **Background work is queue- and cron-driven, not just interval-polled.**
  `task-runner.ts` starts/stops Graphile Worker–backed job processing
  (`storage/graphileWorker.ts`) for queued runs; `schedule-worker.ts` runs
  cron-scheduled robots (`cron-parser`, `node-cron`). `server.ts` also runs
  plain `setInterval`s for queued-run draining, stale browser-slot cleanup,
  and recovering runs stuck in `running` state — don't assume queued runs are
  processed purely by the interval; the Graphile worker is the primary path.
- **`/api` routes are directory-driven, not manually imported.** Every file
  under `server/src/api/` is `readdirSync`'d and `require`d at server start
  and mounted under `/api` (`server.ts`) — adding a new API module means
  dropping a file there with a default (or CommonJS) router export, not
  editing an index/registration list. The older, explicitly-imported routers
  (`webhook`, `record`, `workflow`, `storage`, `auth`, `proxy`) are mounted at
  their own top-level paths instead, from `server/src/routes/`.
- **Auth is session + JWT hybrid.** Express sessions are Postgres-backed
  (`connect-pg-simple`, table auto-created), alongside JWT (`jsonwebtoken`)
  for API auth; `ENCRYPTION_KEY` additionally encrypts stored secrets
  (proxy credentials, integration passwords) at rest.
- **AI/LLM extraction path.** `server/src/sdk/` (`browserAgent.ts`,
  `workflowEnricher.ts`, `selectorValidator.ts`) backs "AI Mode" extraction,
  using `@anthropic-ai/sdk`; `OLLAMA_BASE_URL` is also wired through Docker
  Compose for local-model support.
- **MCP support.** `server/src/mcp-worker.ts` is a separate stdio-transport
  MCP server (`@modelcontextprotocol/sdk`) built via its own tsconfig
  (`server/tsconfig.mcp.json`) and **excluded** from the main server build
  (`server/tsconfig.json`'s `exclude`) — build/run it independently
  (`npm run mcp:build`) rather than expecting it inside `server/dist`.
- **Storage split.** Structured data (robots, runs, users, sessions) lives in
  Postgres via Sequelize; binary run output (screenshots, files) is uploaded
  to MinIO (`storage/mino.ts`-style services) — see the shutdown handler in
  `server.ts` for an example of both being touched together when persisting
  an interrupted run.
- **`legacy/` is dead weight, not a fallback.** It predates the current
  recorder/panel/worker implementations and isn't wired into any build
  target — treat it as historical reference only.

## Conventions

- **Conventional Commits**, format `<type>(<scope>): <short description>`
  (e.g. `feat(api): Add new endpoint for user data`) — see
  `.github/COMMIT_CONVENTION.md` and `CONTRIBUTING.md`.
- **Branch off `develop`**, not `main`/`master` — `develop` is this repo's
  default branch, and PRs target it.
- **AI-assisted contributions are explicitly welcomed** but must be tested,
  understood by the human submitter, and follow existing project structure
  (`CONTRIBUTING.md` §7) — low-quality/unverified AI patches are rejected.
- **TypeScript `strict: true`** in both the root (`tsconfig.json`, frontend)
  and `server/tsconfig.json` (backend) — don't introduce implicit `any`.
- **ESLint config is `eslint-config-react-app`**, declared inline in the root
  `package.json`'s `eslintConfig` (no standalone `.eslintrc*` file) — `npm
  run lint` runs `eslint .` against that config from the repo root.
- **Keep the three `package.json`s in sync.** `package.json` (root, `^`
  ranges, dev workflow) vs. `package.backend.json` / `package.frontend.json`
  (pinned versions, used only for the split Docker images) list overlapping
  but not identical dependency sets — check both when adding/upgrading a
  dependency that either image needs.

## Gotchas

- **No automated test suite exists anywhere in this repo** (root, `server/`,
  `maxun-core/`, `browser/`) despite `maxun-core` declaring a `test: jest`
  script — there's no Jest config or test files backing it, and no CI
  workflow runs it (`.github/` has no `workflows/` directory, only an issue
  template, `COMMIT_CONVENTION.md`, and `CODE_OF_CONDUCT.md`). Don't assume a
  `npm test` you can run to verify a change — manual verification against a
  running instance is the only current option.
- **The root `package.json`'s `start`/`start:dev` scripts run frontend and
  backend concurrently from one process tree** (`concurrently -k ...`) — killing
  one kills both. Use `npm run server:dev` / `npm run client` directly when
  you only want one side running.
- **`server/src/mcp-worker.ts` is intentionally excluded from the main
  backend TypeScript build** (`server/tsconfig.json`'s `exclude` list) — it
  has its own `tsconfig.mcp.json` and `mcp:build` script; forgetting this
  will make an edit there silently not show up in `server/dist`.
- **`/api` route files are auto-discovered by directory listing at server
  boot**, not statically imported — a new file placed under `server/src/api/`
  must export a valid Express router (default or CommonJS export) or the
  server logs `Error: <file> does not export a valid router` and skips it.
- **`maxun-core` is a real dependency boundary, not just an internal
  folder.** Root/backend/frontend `package.json`s all pin a specific
  `maxun-core` version; editing `maxun-core/src` has no effect on a running
  `npm run start`/`start:dev` unless you rebuild and re-link/republish it.
- **Docker Compose images are pulled from `getmaxun/*` by default** — the
  `build:`/`dockerfile:` blocks in `docker-compose.yml` for `backend`,
  `frontend`, and `browser` are commented out in favor of published
  `getmaxun/maxun-backend:latest` / `-frontend:latest` /
  `maxun-browser:latest` images; uncomment the relevant block to test local
  Dockerfile changes instead of the upstream published image.
- **`legacy/` still compiles-adjacent but isn't used** — don't "fix" or
  extend code there under the assumption it's live; find the current
  equivalent in `src/components/recorder/` or
  `server/src/workflow-management/` instead.
