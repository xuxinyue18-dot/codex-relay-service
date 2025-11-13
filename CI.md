# CI (GitHub Actions) Overview

This repository includes a minimal CI workflow under `.github/workflows/ci.yml`.
It runs automatically on push and pull requests targeting `main`.

What it does
- Uses Node.js 18 on Ubuntu runners
- `npm ci` install with npm cache enabled
- Builds Admin SPA: `npm run install:web` and `npm run build:web`
- Lints without auto-fixing: `npm run lint:check`
- Runs tests and passes if no tests exist: `npm test -- --passWithNoTests`
- On pull requests, uploads `web/admin-spa/dist` as a build artifact

How to view results
- Go to the repository → Actions → CI → select the latest run to view logs and artifacts.

Customize
- Node versions: edit `matrix.node-version` (e.g., `[18.x, 20.x]`).
- Skip Admin SPA build: remove the “Install Admin SPA deps / Build Admin SPA” steps.
- Enforce tests: remove `--passWithNoTests` so missing tests fail CI.
- Cache: provided by `actions/setup-node` with `cache: npm`.
- Artifacts: remove the upload step if not needed.

Skip CI for a commit
- Include `[skip ci]` or `[ci skip]` in the commit message subject to skip the workflow.

Rerun and concurrency
- CI cancels in-progress runs for the same ref to save time (see `concurrency` block).
- You can “Re-run all jobs” from the Actions UI.

Secrets (future use)
- Current CI does not require secrets.
- If you add Docker publishing or private registry access, add secrets under
  GitHub → Settings → Secrets and variables → Actions, then reference them in the workflow.

Troubleshooting
- npm install errors: ensure `package-lock.json` is up to date (`npm ci` relies on lockfile).
- Frontend build errors: run `npm run install:web && npm run build:web` locally to reproduce.
- Lint failures: run `npm run lint` locally (or `npm run lint:check` to view issues).

