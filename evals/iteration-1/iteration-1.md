# Iteration 1 — Eval Results

**Setup:** Two parallel subagent runs on eval prompt 1 (audit a missing-everything monorepo). Target repo: `/tmp/opencode/test-repo` (minimal skeleton, no readiness infra). One run used the skill; the other ran on general knowledge alone.

Both completed silently and the target repo was left unchanged — the with-skill run correctly STOPped before implementing. No rogue mutations.

## Headline findings

| Metric                    | With skill                                | Baseline (no skill)                                  |
|---------------------------|--------------------------------------------|------------------------------------------------------|
| Output length             | 100 lines, 9.4 KB                          | 320 lines, 18.9 KB                                   |
| Checklist coverage        | 35 items, all 9 domains, every one verdict | 22 categories invented mid-run, ~40 items, uneven depth |
| Verdict gods              | ✓/✗/⚠ — every item                         | Red/amber implicit, no per-item evidence              |
| STOP gate                 | ✅ fired — "Which domain should I implement?" | n/a — user didn't ask to implement                |
| Cross-run reproducibility | Deterministic (ran checklist commands)     | Stochastic — "second pass added categories 10–22"   |
| Self-reported omissions   | 0 — all 35 items covered                  | baseline self-reported ~15 missing categories post-hoc |

## What the with-skill run validated

- ✅ **Pre-flight step fired**: detected monorepo, workspaces, CJS, no `apps/`, no git remote. Recorded all four.
- ✅ **Audit step fired in quick mode**: ran every detection command from `references/checklist.md` verbatim. All 35 items got a verdict + evidence.
- ✅ **Report step fired**: returned the diffable Markdown table specified in `SKILL.md` §2.
- ✅ **STOP gate fired correctly**: presented priority list, asked "Which domain should I implement?", halted. Implement and Verify correctly did not fire.
- ✅ **No skill contradictions encountered**: agent flagged minor under-specification, not contradictions.

## Skill defects found (fixed in this iteration)

1. **`references/checklist.md` §5 — `.husky/_/` gitignored.** Detection `grep '.husky/_' .gitignore` returned nonzero for two distinct root causes: no `.gitignore` at all, vs `.gitignore` exists but lacks the line. **Fix:** detection now distinguishes three explicit states (no `.gitignore` / missing line / present) and the item notes cross-dependency with the "Repo files" `.gitignore` concerns.

2. **`references/checklist.md` §9 — Branch protection fallback was prose-only, no shell command.** The with-skill agent had to read the prose paragraph and improvise. **Fix:** added explicit verdict rules for HTTP 200 / 404 / "no remote" / "branch absent" so the agent can decide from `gh api` exit output, not prose interpretation.

3. **`SKILL.md` §Pre-flight — didn't explicitly probe `git remote -v`.** Branch-protection items were marked ⚠ on a no-remote repo, but the user wasn't told up-front that they'd be ⚠ until the remote was wired. **Fix:** Pre-flight now records whether a remote exists and flags it to the user at audit time, not at implement time.

## Observations not requiring fixes

- **Baseline went broader than the skill on scope.** The baseline run invented categories like SOC 2, GDPR, pen-test cadence, DR plans, data-retention — out of scope for dev-infra readiness. **Resolution:** added a "Scope" section to README clarifying what this skill does and does not cover. The skill is deliberately narrow; users wanting compliance readiness should layer it separately.

- **Skill SEO/meta detection is Next.js-flavored** (`next-seo|next-sitemap|generateMetadata`). On a Vite frontend, the grep returned empty (correctly), but the agent flagged a stack mismatch. **Resolution:** kept the detection as-is — empty is the correct verdict for a non-Next stack — and the agent's behavior of noting the stack mismatch in evidence is the right pattern. No skill change needed.

- **"Isolated `index.js`" mentioned in the prompt** isn't defined anywhere in the skill. The with-skill agent correctly treated `backend/src/index.js` as the expected entry point, not a finding. No skill change needed — pre-flight shouldn't layer a concept of "isolated files" on top of the layout detection.

## Files changed this iteration

- `SKILL.md` — added git-remote probe to Pre-flight
- `references/checklist.md` — fixed `.husky/_/` gitignore three-state detection; replaced branch-protection prose-only fallback with explicit verdict rules
- `README.md` — added Scope section
- `evals/iteration-1/iteration-1.md` — this report

## Next iteration ideas (not run)

- Run eval prompts 2 and 3 (git-hygiene implement, branch-protection implement) to validate the Implement + Verify steps, not just Audit + STOP.
- Run `skill-creator` description optimizer (`run_loop.py`) against `evals/trigger-eval.json` once a `claude -p`-capable CLI is authenticated — would tune the description for triggering accuracy.