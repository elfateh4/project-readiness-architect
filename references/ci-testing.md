# Testing & CI/CD Implementation Reference

When asked to implement CI/CD or Testing infrastructure, use the blueprints below. Substitute the job `name:` fields to match the repo — `branches:` and `contexts:` arrays reference the same literal branch names owned by `references/repo-files.md` section 8; do not redefine the branch set here.

## 1. CI Quality Gate (GitHub Actions)

Create `.github/workflows/ci.yml`. Matrix covers Backend CI, UI CI, and Commitlint jobs running in parallel; status checks named `ci/backend`, `ci/ui`, and `commitlint` so `repo-files.md` section 8 can reference them in `required_status_checks[contexts]`.

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
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
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
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
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
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v7
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
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
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

Literal branch names owned by `references/repo-files.md` section 8.

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
      - uses: actions/checkout@v7
      - run: docker build -t backend:scan ./backend
      - uses: aquasecurity/trivy-action@0.36.0
        with:
          image-ref: backend:scan
          format: sarif
          output: trivy.sarif
          severity: CRITICAL,HIGH
          exit-code: '0'
      - uses: github/codeql-action/upload-sarif@v4
        if: always()
        with:
          sarif_file: trivy.sarif

  trivy-frontend:
    name: trivy/frontend
    runs-on: ubuntu-latest
    if: hashFiles('frontend/Dockerfile') != ''
    steps:
      - uses: actions/checkout@v7
      - run: docker build -t frontend:scan ./frontend
      - uses: aquasecurity/trivy-action@0.36.0
        with:
          image-ref: frontend:scan
          format: sarif
          output: trivy-frontend.sarif
          severity: CRITICAL,HIGH
          exit-code: '1'
      - uses: github/codeql-action/upload-sarif@v4
        if: always()
        with:
          sarif_file: trivy-frontend.sarif
```

Before pinning, resolve the current trivy-action major via `context7_resolve-library-id`. If newer than `0.30.0`, bump the pin in this reference and in the workflow file written this run.

## 3. Renovate for Dependencies

Create `renovate.json` at the root (or `.github/dependabot.yml` instead — see `references/repo-files.md` section 9; ship exactly one).

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
      - uses: actions/checkout@v7
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
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
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

Lighthouse becomes a required context in `repo-files.md` section 8 only after the CI workflow has run at least once and published the check name `ci/lighthouse` to GitHub — add it to `required_status_checks[contexts]` then.

## 6. Semantic Release workflow

Create `.github/workflows/release.yml`. It runs semantic versioning on every push to `dev`, `staging`, and `main`, producing prerelease tags for dev/staging and a full release for main.

**The agent picks the stack variant** based on the detected stack from pre-flight (JS → Node variant; Python → Python variant; Go → Go variant; Rust → Rust variant). The 3-branch prerelease model (dev=alpha, staging=rc, main=release) is shared across all variants and matches the skill's existing `.releaserc.json` in `references/git-hygiene.md`.

```yaml
name: release

on:
  push:
    branches: [dev, staging, main]

permissions:
  contents: write

jobs:
  release:
    name: release/${{ github.ref_name }}
    runs-on: ubuntu-latest
    concurrency:
      group: release-${{ github.ref_name }}
      cancel-in-progress: false
    steps:
      - name: Checkout
        uses: actions/checkout@v7
        with:
          fetch-depth: 0
          ref: ${{ github.ref_name }}
      - name: Pin branch to workflow sha
        run: git reset --hard ${{ github.sha }}

      # ── Stack-specific step (pick one) ─────────────────────
      # See variants below.
```

### Node / TypeScript variant

Uses `npx semantic-release` with the `.releaserc.json` already configured by `references/git-hygiene.md`. Conventional commits are enforced by the `commitlint` job in `ci.yml` (section 1).

```yaml
      - name: Setup Node
        uses: actions/setup-node@v7
        with:
          node-version-file: .nvmrc
          cache: npm
      - name: Install
        run: npm ci
      - name: Semantic Release
        run: npx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

Version source: `package.json` (`"version"` field). Config: `.releaserc.json`.

### Python / uv variant (leadflow pattern)

Uses `python-semantic-release` with the version read from `pyproject.toml`. Conventional-commit parser is built in.

```yaml
      - name: Setup uv
        uses: astral-sh/setup-uv@v9.0.0
        with:
          version: "0.11.25"
      - name: Python Semantic Release
        uses: python-semantic-release/python-semantic-release@v10.6.1
        with:
          directory: backend
          github_token: ${{ secrets.GITHUB_TOKEN }}
          git_committer_name: "github-actions[bot]"
          git_committer_email: "41898282+github-actions[bot]@users.noreply.github.com"
```

Version source: `pyproject.toml` → `[project] version`. Add this to `backend/pyproject.toml` to configure branches:

```toml
[tool.semantic_release]
version_toml = ["pyproject.toml:project.version"]
commit_parser = "conventional"

[tool.semantic_release.branches.dev]
match = "dev"
prerelease = true
prerelease_token = "alpha"

[tool.semantic_release.branches.staging]
match = "staging"
prerelease = true
prerelease_token = "rc"

[tool.semantic_release.branches.main]
match = "main"
```

### Go variant

Uses `goreleaser` with git tags for versioning. The release is triggered on tag push (a prerelease tag from semantic-release workflow style, or manual `git push origin v*.*.*`). Conventional commits feed into `.goreleaser.yml` changelog generation.

```yaml
      - name: Setup Go
        uses: actions/setup-go@v7
        with:
          go-version-file: go.mod
      - name: GoReleaser
        uses: goreleaser/goreleaser-action@v7.2.3
        with:
          version: '~> v2.14'
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Version source: git tags (`v*.*.*`). Add `.goreleaser.yml` at the repo root to configure snapshot builds on dev/staging:

```yaml
# .goreleaser.yml
builds:
  - env: [CGO_ENABLED=0]
    goos: [linux, darwin, windows]
    goarch: [amd64, arm64]

release:
  prerelease: auto  # dev/staging tags become prereleases; main is full release

changelog:
  sort: asc
  filters:
    exclude: ["^docs:", "^test:", "^ci:"]
```

**3-branch flow for Go:** create the release tags from your branching model — push `v0.0.0-alpha.1` style tags on `dev`, `v0.0.0-rc.1` on `staging`, and `v1.2.3` on `main`. GoReleaser's `prerelease: auto` auto-detects prerelease tags.

### Rust / cargo variant

Uses `release-plz` with a workspace-driven flow. `release-plz` reads version from `Cargo.toml`, publishes to crates.io, and opens/update a release PR on every push.

```yaml
      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
      - name: Run release-plz
        uses: MarcoIeni/release-plz-action@v0.5
        with:
          command: release-pr
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}
```

For the release job (triggered when the release-plz PR merges):

```yaml
  release:
    name: release-plz-release
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v7
      - uses: dtolnay/rust-toolchain@stable
      - name: Run release-plz release
        uses: MarcoIeni/release-plz-action@v0.5
        with:
          command: release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}
```

Version source: `Cargo.toml` (`[workspace.package] version` or per-crate). Add `.release-plz.toml` to configure the 3-branch prerelease model (or use release-plz's built-in `prerelease` setting per branch).

**3-branch flow for Rust:** release-plz's `release-pr` command runs on every push to dev/staging/main, opening a PR with version bumps and changelog. Merge the PR to publish. Use `[package] version = "0.1.0"` style semver; release-plz auto-bumps based on conventional commits.

### Shared rules across all variants

- **Wire semantic-release LAST.** First run on `main` with existing commits creates a real release. If you're adding this to an existing repo, have the user confirm they want that before the first push to `main`.
- **`fetch-depth: 0` is required.** semantic-release and release-plz read the full commit history to compute the next version.
- **Conventional commits are a hard dependency.** The `commitlint` job in `ci.yml` section 1 enforces the format; semantic-release parses it.
- **The concurrency group per branch prevents race conditions.** Two pushes to `dev` simultaneously would otherwise produce conflicting prerelease tags.
- **All action versions MUST be current.** Verify via `context7` MCP before pinning.

**Completion criterion for release workflow:** push a commit to `dev` → the workflow opens a release (or release PR for Rust) without error; check the Actions tab shows green.