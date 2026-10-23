# AGENTS.md

## Project shape

- FastAPI backend: `backend/open_webui` (import path `open_webui`, app `open_webui.main:app`). SvelteKit SPA: `src/`, built into `build/`.
- The backend serves `build/` at `/` when it exists (`FRONTEND_BUILD_DIR` overrides). Without a build it serves API only.
- Root `static/` is the SvelteKit public directory; `backend/open_webui/static` holds backend assets (Swagger UI, logos, loader). Do not confuse the two.
- Version lives in `package.json`; Hatch reads it for the Python package, so bumping one bumps both.
- Ignore `build/`, `.svelte-kit/`, and `node_modules/` — generated output, not source.

## Running it

- Python must be 3.11 or 3.12 (`pyproject.toml` rejects 3.13+). use the `open-webui` conda env (`/home/zhangzeheng/miniforge3/envs/open-webui/bin/python`)
- Backend dev server: `cd backend && ./dev.sh` (uvicorn on `:8080`, `--reload`). It loads the repo-root `.env`. Alembic migrations run automatically on startup (`ENABLE_DB_MIGRATIONS`, default true); SQLite lives in `backend/data/`. Schema changes require a migration in `backend/open_webui/migrations/versions`.
- Frontend dev server: `npm run dev` (`:5173`; `dev:5050` for port 5050). It proxies `/api`, `/ollama`, `/openai`, `/oauth`, `/ws` to `WEBUI_BACKEND_URL` (default `http://localhost:8080`), so run both.
- Production-like check: `npm run build`, then start the backend so it serves `build/`.

## Verification

- `npm run format` — Prettier: tabs, single quotes, 100 cols. CI runs it, then requires `git diff --exit-code`.
- UI strings are extracted into `src/lib/i18n/locales/*/translation.json` by `npm run i18n:parse`. Run it after changing strings and commit the generated catalogs; CI also checks the tree is clean.
- `npm run check` — svelte-check/TypeScript. `npm run lint:frontend` — ESLint with `--fix`.
- Backend formatting is Ruff, not pylint: `ruff format backend` (120 cols, single quotes) and `ruff check --select=F --ignore=F401,F403,F405,F541,F811,F841 backend` mirror CI. Do not rely on `npm run lint`'s `lint:backend` (pylint) step.
- Ruff ICN conventions: `import datetime as dt` and `import ast`-style imports are required; `from datetime import ...` / `from ast import ...` are banned (`pyproject.toml`).
- Tests: only frontend Vitest tests exist. Run one file with `npx vitest run src/lib/components/workspace/common/versionDiff.test.ts`; `npm run test:frontend` uses Vitest watch mode plus `--passWithNoTests`. There is no backend pytest suite here — the backend suite lives in the separate `open-webui/tests` repo and runs only for release PRs.

## Crypto artifact gotcha

- `predev` / `prebuild` run `npm run crypto:verify`. Any change under `crypto/encryption/`, `scripts/encryption/build.py`, or `scripts/encryption/artifact.mjs` makes verification fail until `python scripts/encryption/build.py` regenerates `src/lib/encryption/vendor` (needs Rust, the `wasm32-unknown-unknown` target, `wasm-bindgen`, and network). Never hand-edit that vendor directory.

## Generated assets and config

- `npm run dev` / `npm run build` first run `pyodide:fetch`, which downloads wheels and rewrites `static/pyodide/` plus the tracked `static/pyodide/pyodide-lock.json`. Local diffs there are normal; only commit them for an intentional Pyodide update.
- `.env` is gitignored but loaded by the backend; `.env.example` is the reference. Many settings are seeded into the DB (`ENABLE_PERSISTENT_CONFIG`), so admin-UI/DB values can override `.env` on an existing install.

## Contribution conventions

- Work targets the `dev` branch; PRs against `main` are closed. Code PRs are only accepted when a maintainer requested them (i18n-only PRs are exempt).
- PR titles use prefixes such as `feat:`, `fix:`, `refactor:`, `docs:`, `i18n:`, `build:`, `ci:`, `perf:`, `chore:`.
- Do not add or rewrite tests, fixtures, or testing infrastructure unless explicitly requested.

## Fresh Setup Instructions

### Frontend

```bash
# Only follow this if no build/ is ready
git checkout dev
nvm use 22
# Skip the CUDA pull
ONNXRUNTIME_NODE_INSTALL_CUDA=skip npm install 
npm run build
```
## Backend

```bash
# Only follow this if no backend/.venv nor any conda environment is prepared 
git checkout dev
cp -RpP .env.example .env
key="$(openssl rand -hex 32)"
echo "WEBUI_SECRET_KEY=$key" >> ../.env
cd backend
conda create --name open-webui python=3.11
pip install -r requirements.txt -U
# when build/ is present, fastapi will serve build/
# change the cross_origin dev.sh to * first
sh dev.sh
```

