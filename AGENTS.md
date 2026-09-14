# AGENTS.md

## Layout

- All application code lives in `server/` (TypeScript + Express + Mongoose). There is **no root `package.json`** — run Node commands from `server/`.
- Repo root is Docker orchestration and docs: `docker-compose.yaml` brings up `hae-server` (:3001), `hae-mongo`, and `hae-grafana` (:3000, reads the API via the Infinity datasource using the read token).
- `dashboard-examples/` are importable Grafana dashboards; `docs/` holds images only.

## Commands (from `server/`)

- `yarn dev` — ts-node-dev on `src/app.ts`
- `yarn build` — tsc to `dist/`; `yarn start` — run `dist/app.js`
- `yarn lint` / `yarn lint:fix`, `yarn format`
- **There is no test suite or test runner.** Don't invent one or claim tests pass.
- Full stack from repo root: `sh ./create-env.sh && docker compose up -d`.

## Env / config gotchas

- `server/src/database/mongodb.ts` calls `dotenv.config()`, which reads `.env` from the **current working directory**. The generated `.env` is at repo root, but `yarn dev` runs in `server/`, so root `.env` is *not* loaded locally — export the vars or copy `.env` into `server/`. Docker injects them via compose.
- Required vars: `MONGO_HOST`, `MONGO_USERNAME`, `MONGO_PASSWORD`, `MONGO_DB`, `MONGO_PORT`, `READ_TOKEN`, `WRITE_TOKEN`.
- Auth is an `api-key` header whose value must start with `sk-`. Reads and writes use **separate** tokens (`READ_TOKEN` vs `WRITE_TOKEN`); middleware in `src/middleware/auth.ts`.
- Server does not `exit` cleanly on Mongo connection failure — it calls `process.exit(-1)`.

## API surface

- `POST /api/data` (write token) ingests Health Auto Export JSON of shape `{ data: { metrics?, workouts? } }`. JSON body limit is 200mb.
- `GET /api/metrics/:selected_metric`, `GET /api/workouts`, `GET /api/workouts/:id` (read token).
- `:selected_metric` must be a `MetricName` enum value (`src/models/MetricName.ts`); that string becomes the Mongo collection name.
- `blood_pressure`, `heart_rate`, and `sleep_analysis` have dedicated schemas in `src/models/Metric.ts`; every other metric uses the generic base schema. Add new special shapes there.
- Ingestion is idempotent via upserts: metrics on `{ source, date }`, workouts on `workoutId`. Don't change these keys without a migration.

## Style

- ESLint enforces `import/order`: builtin → external → internal → parent/sibling → index, alphabetized, blank line between groups.
- Prettier is wired through ESLint (`plugin:prettier/recommended`); single quotes, trailing commas, print width 100.

## Role in Nick's health stack (2026-09-13)

Runs on nlc-server (Proxmox Docker LXC; global skill `nlc-server`). The iPhone Health Auto Export app posts to `POST /api/data`. BioHackz's `scripts/import-apple-health.cjs` will gain an `--from-hae <url>` source reading `GET /api/metrics/:metric` with the read token instead of `export.xml`; that is a follow-up plan in the BioHackz repo.
