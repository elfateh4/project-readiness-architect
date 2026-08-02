# Repo Files & Branch Protection Implementation Reference

When asked to implement repo files or branch protection, use the blueprints below. All paths are repo-relative.

## 1. LICENSE

If the user has no preferred license, default to **MIT** (permissive, common). Reuse this template — substitute `<year>` and `<holder>` from `git config user.name` and the current year.

```
MIT License

Copyright (c) <year> <holder>

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

Ask the user before writing if they prefer Apache-2.0, GPL-3.0, or proprietary. **Idempotency:** if `LICENSE` exists, do not overwrite — read it and diff against the template; report the divergence, don't clobber.

## 2. SECURITY.md

`.github/SECURITY.md`:

```markdown
# Security Policy

## Reporting a Vulnerability

Please report security issues to security@<your-domain>. Do NOT open a public GitHub issue.

- Acknowledgement within 48 hours.
- A fix or mitigation timeline within 7 days.
- We credit responsible reporters in release notes (opt-in — let us know your preferred handle).

## Supported Versions

| Version | Supported          |
|---------|--------------------|
| latest main | :white_check_mark: |
| older      | :x:                |

## Disclosure

We follow coordinated disclosure: a CVE may be filed once a fix is released.
```

Substitute `<your-domain>` with the user's domain. If none, leave a placeholder and flag it to the user at the end of the run.

## 3. CODEOWNERS

`.github/CODEOWNERS`:

```
# Default — request review from the platform team for anything not covered below.
* @<platform-team-handle>

# Backend
/backend/ @<backend-team-handle>

# Frontend
/frontend/ @<frontend-team-handle>

# CI / workflows / repo tooling
/.github/ @<platform-team-handle>
/.husky/ @<platform-team-handle>
```

Ask the user for the team handles (or individual handles). Leave placeholders and flag at the end if none provided.

## 4. .env.example + .gitignore

Append to `.gitignore` (idempotent — only append lines that aren't already there):

```
# Secrets
.env
.env.*
!.env.example
```

Create `.env.example` with every key the app reads from env, sourced by scanning `process.env.KEY` / `import.meta.env.VITE_KEY` in the repo. Leave values blank or with sensible dev-defaults **only for non-secret config** (ports, log-level). Secret keys (DB password, JWT secret, API tokens) get a placeholder like `<set-in-CI>`.

Template:

```env
# App
NODE_ENV=development
PORT=9000
LOG_LEVEL=info

# Database
DATABASE_URL=postgres://postgres:postgres@localhost:5432/app

# Secrets — set in CI / production, never in this file
JWT_SECRET=<set-in-CI>
API_KEY=<set-in-CI>
```

After writing, grep the repo one more time for `process.env.` and ensure every referenced key appears in `.env.example`. **Completion criterion:** `grep -rhoE "process\.env\.[A-Z_]+" backend frontend 2>/dev/null | sort -u` returns a set fully covered by the example file.

## 5. .editorconfig

`.editorconfig`:

```ini
root = true

[*]
indent_style = space
indent_size = 2
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.md]
trim_trailing_whitespace = false

[*.{yml,yaml}]
indent_size = 2

[Makefile]
indent_style = tab
```

## 6. .nvmrc

Resolve the current LTS via the `context7` MCP (`context7_resolve-library-id` for Node release info) or fall back to the latest LTS codename at run time. Write the major version only:

```
20
```

If the user uses a different runtime manager (`.tool-versions` for asdf/volta), write that file instead:

```
nodejs 20.18.0
```

## 7. Branch Protection via `gh`

Requires `gh` authenticated against the repo remote. If `gh auth status` fails, STOP and tell the user to run `gh auth login` — do not attempt to set protection via raw API tokens manipulated by the agent.

The 3-branch model: `dev` (working), `staging` (release candidate), `main` (production). Single source of truth — the literal branch names live here; `.releaserc.json` and workflow `on:` branches reference these names but don't redefine them.

For each branch (`dev`, `staging`, `main`):

```bash
gh api \
  --method PUT \
  repos/:owner/:repo/branches/<branch>/protection \
  -F required_status_checks[strict]=true \
  -F required_status_checks[contexts][]="ci/backend" \
  -F required_status_checks[contexts][]="ci/ui" \
  -F required_status_checks[contexts][]="commitlint" \
  -F enforce_admins=true \
  -F required_pull_request_reviews[required_approving_review_count]=1 \
  -F required_pull_request_reviews[dismiss_stale_reviews]=true \
  -F required_pull_request_reviews[require_code_owner_reviews]=true \
  -F restrictions= \
  -F allow_force_pushes=false \
  -F allow_deletions=false
```

**For `main` only:** add `-F required_pull_request_reviews[required_approving_review_count]=2` and `-F required_linear_history=true`.

**Contexts must match the actual job names in `.github/workflows/ci.yml`.** After writing the CI workflow (see `references/ci-testing.md`), read its job `name:` fields and substitute the exact strings into the `required_status_checks[contexts]` array before running this command. Mismatched context names produce a silent red box in PRs.

Verify after each:

```bash
gh api repos/:owner/:repo/branches/<branch>/protection | head -5
```

A 200 response means protected; a 404 means unprotected. Paste each branch's status into the report.

## 8. Renovate alternative (optional)

If the user prefers Dependabot over Renovate (or Renovate is disabled at the org level), create `.github/dependabot.yml` instead of `renovate.json`:

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule: { interval: "weekly" }
    open-pull-requests-limit: 5
    groups:
      minor-patch:
        update-types: ["minor", "patch"]
  - package-ecosystem: "docker"
    directory: "/backend"
    schedule: { interval: "weekly" }
```

Do not ship both `renovate.json` and `.github/dependabot.yml` — pick one and document the choice in the report.