# Readiness Checklist with Detection Rules

Every item below carries: **what "ready" means** and **how to detect it**. Verdicts: ✓ Present, ✗ Missing, ⚠ Partial (files exist but not wired). The detection rule is the source of truth for the audit — run it, don't guess.

## 1. Planning & Architecture *(audit-only by default)*

| Item | Ready means | Detection (shell) |
|------|-------------|-------------------|
| Sitemap defined | `sitemap.xml` or a route map under `docs/` | `test -f public/sitemap.xml \|\| test -d docs && ls docs/ \| grep -i sitemap` |
| Design token system | Style Dictionary → CSS vars (or tokens repo) | `ls packages/ui/tokens/ 2>/dev/null; grep -r "style-dictionary" package.json packages/*/package.json 2>/dev/null` |

## 2. Frontend *(audit-only by default)*

| Item | Ready means | Detection |
|------|-------------|-----------|
| Atomic design structure | atoms → molecules → organisms dirs | `ls frontend/src/web/components/{atoms,molecules,organisms} 2>/dev/null \|\| find frontend/src -type d \| grep -E 'atoms\|molecules\|organisms'` |
| Storybook set up | `.storybook/` + `storybook` dep + npm script | `test -d .storybook; grep '"storybook"' package.json; grep '"storybook":' package.json scripts` |
| Global SEO & meta | next-seo / next-sitemap / `metadata` API used | `grep -rE "next-seo\|next-sitemap\|generateMetadata" frontend/src 2>/dev/null` |
| Performance/SEO/a11y budgets | `.lighthouserc.json` with assertions | `test -f .lighthouserc.json && grep -A3 assertions .lighthouserc.json` |

## 3. Backend

| Item | Ready means | Detection |
|------|-------------|-----------|
| Feature/app folder structure | `backend/src/features/<feat>/` or `apps/<mod>/` | `ls backend/src/features 2>/dev/null \|\| ls backend/src/apps 2>/dev/null` |
| OpenAPI spec maintained/auto-gen | `docs/openapi.yaml` + `npm run openapi` script | `test -f docs/openapi.yaml; grep '"openapi"' backend/package.json` |
| Structured logging (Pino) | pino dep + `logger.js` exporting pino instance | `grep -l pino backend/package.json; test -f backend/src/common/lib/logger.js` |

## 4. Testing

| Item | Ready means | Detection |
|------|-------------|-----------|
| Unit tests (Vitest/Jest) | dep installed + `*.test.ts` present + run script | `grep -E '"vitest"\|"jest"' package.json backend/package.json 2>/dev/null; find . -name '*.test.ts' -not -path '*/node_modules/*' \| head -3` |
| E2E tests (Playwright) | `playwright.config.*` + `tests/e2e/` | `test -f playwright.config.ts; ls tests/e2e/ 2>/dev/null` |
| API mocking (MSW) | `msw` dep + `handlers.ts` | `grep '"msw"' frontend/package.json; find frontend/src -name 'handlers.*'` |
| Load testing (k6) | `tests/load/*.js` + k6 in CI | `ls tests/load 2>/dev/null; grep -r "k6" .github/workflows 2>/dev/null` |

## 5. Git & Repo Hygiene

| Item | Ready means | Detection |
|------|-------------|-----------|
| 3-branch model | `dev`, `staging`, `main` all exist | `git branch -a \| grep -E 'dev\|staging\|main'` |
| Conventional Commits (commitlint) | `commitlint.config.*` + dep | `ls commitlint.config.* 2>/dev/null; grep commitlint package.json` |
| Git hooks (husky) | `.husky/` dir + `prepare: husky` script | `test -d .husky; grep '"prepare"' package.json` |
| pre-commit: lint + format | `.husky/pre-commit` runs lint-staged | `test -f .husky/pre-commit && grep lint-staged .husky/pre-commit; test -f .lintstagedrc.json` |
| Secret scanning (gitleaks) | hook calls gitleaks OR gitleaks in CI | `grep gitleaks .husky/pre-commit .github/workflows/*.yml 2>/dev/null` |
| Semantic versioning + changelog | `.releaserc.json` + `CHANGELOG.md` | `test -f .releaserc.json; test -f CHANGELOG.md` |
| Husky internal dir gitignored | `.husky/_/` in `.gitignore` | `test -f .gitignore && grep -q '.husky/_' .gitignore` — distinguish three states: no `.gitignore` at all (✗, blocks all other gitignore-dependent items), `.gitignore` exists but lacks the line (✗, fix is one append), or line present (✓) |

## 6. CI/CD

| Item | Ready means | Detection |
|------|-------------|-----------|
| Build Docker image & push ghcr.io | workflow builds + pushes to `ghcr.io` | `grep -rl "ghcr.io" .github/workflows 2>/dev/null` |
| Docker vuln scanning (Trivy) | `security.yml` uses trivy-action | `grep -rl "trivy-action" .github/workflows 2>/dev/null` |
| Automated deps (Renovate) | `renovate.json` at root | `test -f renovate.json` |
| Static analysis gate | CI job runs lint/typecheck/test as required check | `grep -E "lint\|typecheck\|test" .github/workflows/ci.yml` |

## 7. Local Dev Experience

| Item | Ready means | Detection |
|------|-------------|-----------|
| Makefile for common commands | `Makefile` with `dev`, `test`, `lint`, `build` targets | `test -f Makefile; grep -E '^dev:|^test:|^build:' Makefile` |
| docker-compose full local stack | `docker-compose.yml` with app + deps | `test -f docker-compose.yml \|\| test -f compose.yaml` |

## 8. Repo Files

| Item | Ready means | Detection |
|------|-------------|-----------|
| LICENSE | `LICENSE` file present | `test -f LICENSE` |
| SECURITY.md | `SECURITY.md` present | `test -f SECURITY.md` |
| CODEOWNERS | `.github/CODEOWNERS` present | `test -f .github/CODEOWNERS` |
| `.env.example` committed + `.env` ignored | example present + .env in gitignore | `test -f .env.example; grep -E '^\.env$' .gitignore` |
| `.editorconfig` | committed | `test -f .editorconfig` |
| `.nvmrc` / `.tool-versions` | node version pinned for devs | `test -f .nvmrc \|\| test -f .tool-versions` |

## 9. Branch Protection

| Item | Ready means | Detection |
|------|-------------|-----------|
| `dev` protected | require PR + status checks on dev | `gh api repos/:owner/:repo/branches/dev/protection` |
| `staging` protected | same | `gh api repos/:owner/:repo/branches/staging/protection` |
| `main` protected | require PR + status checks + no force-push | `gh api repos/:owner/:repo/branches/main/protection` |

Verdict rules for the `gh api .../protection` check:
- HTTP 200 → ✓ protected.
- HTTP 404 → ✗ unprotected (branch exists remotely, no protection rule).
- "Could not resolve to a Repository" / no `origin` remote → ⚠ (cannot verify remotely; first run `git remote add origin <url>` and `git push -u origin dev staging main`, then re-audit).
- Branch absent (`git branch -a | grep -q staging` returns 1) → ✗ (branch doesn't exist yet — branch protection can't apply; create it before fixing).

Pre-flight for this section: `git remote -v` must show at least one `fetch` remote. If it doesn't, all three branch-protection items are ⚠ and the next step is wiring the remote, not setting protection.