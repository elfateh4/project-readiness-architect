# Git & Repo Hygiene Implementation Reference

When asked to implement Git Hygiene, set up Husky, Commitlint, Lint-staged, and Semantic Release. Configure for **ESM by default** — modern versions of all four packages are ESM-first. If the root `package.json` has `"type": "commonjs"` deliberately, stop and ask the user before flipping to `"module"`.

## 0. Module-system decision

Add `"type": "module"` to the root `package.json` (if not already present) so the config files below load. If the user's stack mandates CJS, use the `.cjs` variants noted beside each file.

## 1. Root `package.json`

Add the necessary dependencies and scripts:

```json
{
  "type": "module",
  "scripts": {
    "prepare": "husky"
  },
  "devDependencies": {
    "@commitlint/cli": "^21.2.1",
    "@commitlint/config-conventional": "^21.2.0",
    "husky": "^9.1.7",
    "lint-staged": "^17.3.0",
    "semantic-release": "^25.0.9",
    "@semantic-release/changelog": "^7.0.0",
    "@semantic-release/exec": "^7.1.0",
    "@semantic-release/git": "^11.0.1",
    "@semantic-release/github": "^12.0.9",
    "@semantic-release/commit-analyzer": "^13.0.1",
    "@semantic-release/release-notes-generator": "^14.1.1"
  }
}
```

Before pinning, resolve the current major of `husky`, `commitlint`, `semantic-release` via the `context7` MCP (`context7_resolve-library-id` → `context7_query-docs`). If a newer major exists, update the pin in this file and in the installed `package.json` for this run; flag the bump to the user.

## 2. Commitlint Config

Create `commitlint.config.mjs` at the root:

```js
export default {
  extends: ['@commitlint/config-conventional'],
};
```

CJS fallback: `commitlint.config.cjs` → `module.exports = { extends: ['@commitlint/config-conventional'] };`

## 3. Husky Hooks

Husky v9 hooks no longer need the shebang line.

Create `.husky/commit-msg`:

```sh
npx --no -- commitlint --edit "$1"
```

Create `.husky/pre-commit`:

```sh
npx lint-staged

# Optional secret scanning — only runs if gitleaks is installed locally
if command -v gitleaks &>/dev/null; then
  gitleaks protect --staged -v
fi
```

Husky v9 hooks do not need to be made executable (the `npx husky` init marks the dir executable); to be safe on hardened systems run:

```bash
chmod +x .husky/commit-msg .husky/pre-commit
```

## 4. Lint-staged Config

Create `.lintstagedrc.json` at the root. **Substitute the globs to match the detected layout** from the pre-flight step (`SKILL.md` section Pre-flight). The globs below assume the canonical monorepo (`frontend/src/web`, `frontend/src/admin`, `frontend/packages/ui`, `backend/`).

```json
{
  "frontend/src/web/**/*.{js,jsx}":     ["npx eslint --fix"],
  "frontend/src/admin/**/*.{js,jsx}":  ["npx eslint --fix"],
  "frontend/packages/ui/**/*.{ts,tsx}": ["npx biome check --write"],
  "backend/**/*.js":                   ["npx eslint --fix"]
}
```

If a detected dir doesn't exist (e.g. no `frontend/src/admin/`), drop that line — never lint a glob whose target is empty.

Append to `.gitignore` (idempotent — only append lines not already present):

```
# Husky internals
.husky/_/
```

## 5. Semantic Release

Create `.releaserc.json` at the root. The literal branch names (`main`, `staging`, `dev`) are the single source of truth — `references/repo-files.md` section 8 and `references/ci-testing.md` section 2 reference the same names, don't redefine them here. This config is consumed by `.github/workflows/release.yml` (see `references/ci-testing.md` section 6).

```json
{
  "branches": [
    "main",
    { "name": "staging", "prerelease": "rc" },
    { "name": "dev", "prerelease": "alpha" }
  ],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    [
      "@semantic-release/exec",
      {
        "prepareCmd": "printf '%s' '${nextRelease.version}' > VERSION"
      }
    ],
    [
      "@semantic-release/git",
      {
        "assets": ["CHANGELOG.md", "VERSION"],
        "message": "chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
      }
    ],
    "@semantic-release/github"
  ]
}
```

Create `CHANGELOG.md` **seeded with the header** that semantic-release appends after:

```markdown
# Changelog

All notable changes to this project will be documented in this file. See [semantic-release](https://github.com/semantic-release/semantic-release) for commit conventions.
```

**Idempotency:** if `CHANGELOG.md` already exists with content beyond the header, leave it — semantic-release will append. If it exists empty or with only the header, ensure the header is present and leave alone.

## 6. Idempotency check before claiming done

If any of `commitlint.config.*`, `.husky/*`, `.lintstagedrc.json`, `.releaserc.json`, `CHANGELOG.md` already exist at the start of this run, diff the new content against the existing file and merge. Report which files were created vs. merged. A re-run on an already-ready repo must produce "no changes" for every file.