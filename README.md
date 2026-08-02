# project-readiness-architect

[Live on skills.sh](https://skills.sh/elfateh4/project-readiness-architect) · [GitHub](https://github.com/elfateh4/project-readiness-architect)

> An agent skill that audits a repository against a production-readiness checklist and implements whatever's missing — git hygiene, CI/CD, testing, OpenAPI, repo files, branch protection.

A single agent skill (`SKILL.md` + bundled references) that turns "is my repo ready for production?" into a deterministic workflow: **Audit → Report → STOP → Implement one domain → Verify**. Built for the open agent skills ecosystem and installable via the `skills` CLI.

## Why

Most "production readiness" advice is a checklist blog post — the human still has to wire husky, write the commitlint config, fix the `ERR_REQUIRE_ESM` crash in their pino script, match CI job names to `required_status_checks[contexts]`, and remember `.husky/_/` goes in `.gitignore`. This skill does that work. It runs detection rules (`test -f`, `grep`, `gh api`) instead of vibes, so re-running the audit on the same repo produces the same verdict.

## What it covers

Nine domains, each with detection rules in [`references/checklist.md`](references/checklist.md):

1. Planning & Architecture — sitemap, design tokens
2. Frontend — atomic design, Storybook, SEO/meta, Lighthouse budgets
3. Backend — feature folders, OpenAPI auto-gen, structured logging (Pino)
4. Testing — Vitest, Playwright, MSW, k6
5. Git & Repo hygiene — 3-branch model, commitlint, husky, lint-staged, gitleaks, semantic-release
6. CI/CD — Docker → ghcr.io, Trivy, Renovate, static analysis gate
7. Local Dev Experience — Makefile, docker-compose
8. Repo files — LICENSE, SECURITY.md, CODEOWNERS, `.env.example`, `.editorconfig`, `.nvmrc`
9. Branch protection — `dev` / `staging` / `main` via `gh api`

Domains 1, 2, 7 audit-only by default (stack- and taste-dependent). The rest the skill implements from the reference blueprints.

## How it works

```
Pre-flight  →  Audit  →  Report  →  STOP  →  Implement  →  Verify
```

- **Pre-flight** probes the repo layout (monorepo vs. single-app) so the globs in the reference blueprints match your actual tree, not a hardcoded `frontend/src/web/`.
- **Audit** runs detection rules from `references/checklist.md` and marks every item ✓ / ✗ / ⚠ with one line of evidence. Quick mode is `test -f` + `grep`; deep mode invokes the tools in dry-run (`semantic-release --dry-run`, `gh api …/protection`).
- **STOP** — the skill will not implement anything until you pick which missing domain to address. No autonomous fires into the repo.
- **Implement** runs one domain at a time, idempotently — re-running on an already-ready repo is a no-op. Version pins resolve to current majors via the `context7` MCP before writing.
- **Verify** demands evidence (exit-0 commands, actual `gh api` responses, a non-empty `docs/openapi.yaml`) before claiming done.

## Scope

This skill covers **dev-infra readiness** — the repo hygiene, CI/CD, testing, build, and release machinery an engineering team needs before shipping. It does **not** cover enterprise procurement / compliance readiness (SOC 2, GDPR, pen-test cadence, incident-response plans, DR, data-retention, accessibility VPATs, observability stack). Those are out of scope and should be layered on separately (e.g., via a compliance consultancy or a SOC 2 readiness tool). If your definition of "production-ready" includes those, run this skill first to get the engineering baseline, then layer the compliance work on top.

## Install

```bash
npx skills add elfateh4/project-readiness-architect -g -y
```

The `-g` installs globally (user-level); drop it for project-scoped. After install, the skill fires automatically when you ask your agent things like:

- "audit my repo for production readiness"
- "set up commitlint and husky on this monorepo"
- "wire GitHub Actions to build and push my Docker image to ghcr.io"
- "what's missing before we can ship to prod?"

## Use

Run a one-shot audit:

```
Audit this repo against the Project Readiness Checklist and report what's missing.
```

The skill will return a diffable table — every checklist item, ✓ / ✗ / ⚠, evidence — followed by a priority-ordered list of what to fix. Then it stops and asks which domain to implement first.

## Repository layout

```
project-readiness-architect/
├── SKILL.md                          # workflow + index, ~100 lines
├── LICENSE
├── README.md
├── evals/
│   └── evals.json                    # 3 realistic test prompts
└── references/
    ├── checklist.md                  # 9 domains × (item, ready-means, detection-rule)
    ├── git-hygiene.md                # husky, commitlint, semantic-release (ESM)
    ├── ci-testing.md                 # ci.yml, security.yml, k6, lighthouse, renovate
    ├── backend-readiness.md          # pino, zod→openapi (ESM, registry completeness check)
    └── repo-files.md                 # LICENSE/SECURITY/CODEOWNERS/.env.example/.editorconfig/.nvmrc + branch protection
```

All reference files are progressive disclosure — the agent reads them only when implementing that domain, so `SKILL.md` stays light.

## Requirements

The implementing agent should have:

- **Bash** for `test`, `grep`, `git`, `gh`
- **`gh` authenticated** for branch protection (`gh auth login`)
- **`context7` MCP** (optional but recommended) for version-pin freshness checks
- **Node.js** with npm for installing husky / commitlint / semantic-release / pino / zod-to-openapi

The skill does not install Node or `gh` itself; it verifies their presence in the Pre-flight step and stops if absent.

## Design notes

- **Single source of truth** for the literal branch names (`dev`, `staging`, `main`) — owned by `references/repo-files.md` §7; everything else references, never redefines.
- **Idempotent by design** — every implementation file is diffed and merged if it already exists. A re-run on an already-ready repo is a no-op.
- **Stack-aware** — the pre-flight step probes the monorepo shape and substitutes globs. The reference blueprints assume a canonical `frontend/` + `backend/` layout; if your layout is different, the skill adapts.
- **ESM by default** — all reference code uses `import`/`export`. CJS fallbacks are noted inline for stacks that mandate it. The skill no longer ships configs that crash with `ERR_REQUIRE_ESM` on a modern `commitlint`/`pino`/`zod-to-openapi` install.

## License

MIT. See [`LICENSE`](LICENSE).

## Contributing

Issues and PRs welcome at <https://github.com/elfateh4/project-readiness-architect>. The skill is most useful when its reference blueprints track current majors of husky, commitlint, semantic-release, pino, zod-to-openapi, trivy-action, and k6 — pin-drift is the main source of sediment, so any "bump pin to vX" PR is appreciated.