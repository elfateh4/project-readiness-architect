# Testing & CI/CD Implementation Reference

When asked to implement CI/CD or Testing infrastructure, use the blueprints below. Substitute the job `name:` fields to match the repo — `branches:` and `contexts:` arrays reference the same literal branch names owned by `references/repo-files.md` §7; do not redefine the branch set here.

## 1. CI Quality Gate (GitHub Actions)

Create `.github/workflows/ci.yml`. Matrix covers Backend CI, UI CI, and Commitlint jobs running in parallel; status checks named `ci/backend`, `ci/ui`, and `commitlint` so `repo-files.md` §7 can reference them in `required_status_checks[contexts]`.

```yaml
name: ci

on:
  push:
    branches: [dev, staging, main]
  pull_request:
    branches: [dev, staging, main]

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  backend:
    name: ci/backend
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm
          cache-dependency-path: backend/package-lock.json
      - run: npm ci
        working-directory: backend
      - run: npm run lint --if-present
        working-directory: backend
      - run: npm run typecheck --if-present
        working-directory: backend
      - run: npm test -- --coverage
        working-directory: backend

  ui:
    name: ci/ui
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm
          cache-dependency-path: frontend/package-lock.json
      - run: npm ci
        working-directory: frontend/src/web
      - run: npm run lint --if-present
        working-directory: frontend/src/web
      - run: npm run typecheck --if-present
        working-directory: frontend/src/web
      - run: npm run test -- --run
        working-directory: frontend/src/web
      - run: npm run build
        working-directory: frontend/src/web

  commitlint:
    name: commitlint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm
      - run: npm ci
      - run: npx commitlint --from ${{ github.event.pull_request.base.sha || github.event.before }} --to ${{ github.event.pull_request.head.sha || github.sha }}

  e2e:
    name: ci/e2e
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npx playwright test
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7
```

If the repo is single-app (no `backend/` or `frontend/src/web/`), collapse the relevant job's `working-directory:` and `cache-dependency-path:` to `/`. If a job's working dir doesn't exist after pre-flight detection, drop that job — but record it as excluded in the report.

## 2. Security Scanning (Trivy)

Literal branch names owned by `references/repo-files.md` §7.

Create `.github/workflows/security.yml`:

```yaml
name: security

on:
  push:
    branches: [dev, staging, main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'

permissions:
  security-events: write

jobs:
  trivy-backend:
    name: trivy/backend
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t backend:scan ./backend
      - uses: aquasecurity/trivy-action@0.30.0
        with:
          image-ref: backend:scan
          format: sarif
          output: trivy.sarif
          severity: CRITICAL,HIGH
          exit-code: '0'
      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy.sarif

  trivy-frontend:
    name: trivy/frontend
    runs-on: ubuntu-latest
    if: hashFiles('frontend/Dockerfile') != ''
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t frontend:scan ./frontend
      - uses: aquasecurity/trivy-action@0.30.0
        with:
          image-ref: frontend:scan
          format: sarif
          output: trivy-frontend.sarif
          severity: CRITICAL,HIGH
          exit-code: '1'
      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-frontend.sarif
```

Before pinning, resolve the current trivy-action major via `context7_resolve-library-id`. If newer than `0.30.0`, bump the pin in this reference and in the workflow file written this run.

## 3. Renovate for Dependencies

Create `renovate.json` at the root (or `.github/dependabot.yml` instead — see `references/repo-files.md` §8; ship exactly one).

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "schedule": ["every weekend"],
  "prHourlyLimit": 5,
  "packageRules": [
    {
      "matchUpdateTypes": ["minor", "patch"],
      "matchCurrentVersion": "!/^0/",
      "automerge": true,
      "automergeType": "pr",
      "platformAutomerge": true
    }
  ]
}
```

## 4. Load Testing (k6)

Create `tests/load/k6-smoke.js` for quick API sanity checks:

```js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');
export const options = {
  scenarios: { smoke: { executor: 'constant-vus', vus: 3, duration: '30s' } },
  thresholds: { http_req_duration: ['p(95)<500'], errors: ['rate<0.05'] },
};

export default function () {
  const res = http.get(__ENV.BASE_URL || 'http://localhost:9000/health');
  errorRate.add(!check(res, { '200': (r) => r.status === 200 }));
  sleep(0.1);
}
```

Add a CI job (append to `.github/workflows/security.yml` or a new `load.yml`) that runs this against the staging deployment on schedule:

```yaml
  k6-smoke:
    name: k6/smoke
    runs-on: ubuntu-latest
    on:
      schedule: [{ cron: '0 */6 * * *' }]
    steps:
      - uses: actions/checkout@v4
      - uses: grafana/k6-action@v0.3.1
        with:
          filename: tests/load/k6-smoke.js
        env:
          BASE_URL: https://staging.example.com
```

## 5. Lighthouse CI

Create `.lighthouserc.json` in the root. The URL and port assume the frontend's `vite preview` default (`4173`); substitute the actual preview port for the detected framework.

```json
{
  "ci": {
    "collect": {
      "url": ["http://localhost:4173/"],
      "startServerCommand": "cd frontend/src/web && npx vite preview --port 4173",
      "startServerReadyPattern": "Local",
      "startServerReadyTimeout": 30000
    },
    "assert": {
      "preset": "lighthouse:recommended",
      "assertions": {
        "categories:accessibility": ["error", { "minScore": 0.9 }],
        "categories:seo": ["error", { "minScore": 0.9 }],
        "categories:performance": ["warn", { "minScore": 0.8 }]
      }
    }
  }
}
```

Add a CI job to `.github/workflows/ci.yml` (only on PRs touching `frontend/`):

```yaml
  lighthouse:
    name: ci/lighthouse
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm
          cache-dependency-path: frontend/src/web/package-lock.json
      - run: npm ci
        working-directory: frontend/src/web
      - run: npm run build
        working-directory: frontend/src/web
      - run: npx @lhci/cli autorun
```

Lighthouse becomes a required context in `repo-files.md` §7 only after the CI workflow has run at least once and published the check name `ci/lighthouse` to GitHub — add it to `required_status_checks[contexts]` then.