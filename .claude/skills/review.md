---
name: review
description: >
  Review code changes — a branch, a PR, staged changes, or specific files —
  for correctness, security, design, quality, performance, and test
  coverage. Use when asked to "review", "check this diff", "give feedback",
  "review this PR", or "post a review". Classifies risk, dispatches
  specialized agents in parallel, and aggregates results into one summary.
---

# Code Review

<!--
SCAFFOLD NOTES (delete once adapted):
- This skill orchestrates the specialized reviewers in `.claude/agents/`
  (correctness, security, architecture, code-quality, performance, test).
  It doesn't duplicate their checklists — see each agent file for what it
  actually looks for.
- Replace every `REPLACE_ME` with project-specific commands/paths.
- This implements "Level 3: multi-agent, risk-tiered review" from the team's
  maturity ladder for code review. It covers two of the six layers of
  checks a mature pipeline needs: pre-PR verification and automated AI
  review. Required CI, a dedicated security gate, human approval, and the
  merge queue are separate layers this skill assumes exist alongside it —
  don't treat a clean run of this skill as a substitute for them.
-->

The primary goal is **correctness**. Read the change with a critical eye for
logic errors, bad state handling, and failure modes — the specialized agents
below are a checklist of common patterns, not a replacement for actually
understanding whether the change does what it claims.

## Step 1: Identify the change

Figure out what to review from context:

- **Branch vs. trunk**: diff against the merge base, not just the branch
  tip, so the review covers only what this branch actually introduces:
  ```bash
  trunk=REPLACE_ME   # e.g. origin/main
  merge_base=$(git merge-base HEAD "$trunk")
  git log --oneline "$merge_base"..HEAD
  git diff "$merge_base"..HEAD
  ```
- **Staged changes**: `git diff --cached`.
- **A pull request**: `gh pr diff <number>` and `gh pr view <number>` for
  the description and linked issue. If you need to read files at the PR's
  exact version, `gh pr checkout <number>` — but check `git status` first
  and ask before checking out over uncommitted local changes.
- **Specific files**: whatever the user named directly.

Read enough surrounding, unchanged code to understand context. Don't judge
a diff in isolation.

## Step 2: Classify risk

Before dispatching every agent on every change, size up the risk:

- **Trivial** (docs/comments only, formatting, dependency version bumps
  with no behavior change, generated-file updates): a quick read is enough.
  Skip the full agent dispatch; just confirm nothing unexpected slipped in.
- **Low** (small, localized, well-tested, no security/data surface):
  run `correctness-reviewer` and `code-quality-reviewer`; skip the rest
  unless the diff touches something in their trigger conditions below.
- **Medium/high** (touches auth, data access, public APIs, concurrency,
  migrations, or a hot path; or is large/cross-cutting): run every
  applicable agent from the table below.

State which tier you assigned and why in the final summary — don't apply
this silently. If the project has its own risk policy (e.g. a `REVIEW.md`
per area, or rules for what can auto-approve), defer to it over this
default.

## Step 3: Select review aspects

Each aspect is a specialized agent in `.claude/agents/`. Not every aspect
applies to every change — check the diff against the trigger column and
skip what doesn't apply.

| Aspect | Agent | Applies when |
|--------|-------|--------------|
| Correctness & safety | `correctness-reviewer` | Always, for any non-trivial change |
| Security | `security-reviewer` | Touches auth, input handling, secrets, crypto, or external data |
| Architecture & design | `architecture-reviewer` | Adds/changes a module boundary, public interface, or abstraction |
| Code quality & comments | `code-quality-reviewer` | Always — cheap, high-signal pass |
| Performance | `performance-reviewer` | Touches a hot path, data access code, or loops over external data |
| Test coverage | `test-reviewer` | Test files changed, or new behavior added without tests |

The user can override selection:
- `/review` — run every applicable aspect (default).
- `/review security performance` — run only the named aspects.
- Add project-specific aspects here if you introduce more agents.

## Step 4: Dispatch agents

**Default to parallel.** Launch every applicable agent at once. Run
`correctness-reviewer` in the foreground — it needs to read the most
surrounding context and takes longer than the others. Launch the rest in
the background first, then do the foreground correctness pass while they
run.

**Don't poll for background results.** Background reviews often take a
couple of minutes; rely on the automatic completion notification instead
of repeatedly checking. Wait for all notifications before aggregating.

**Vary the reviewing model from the authoring model when you can.** A
reviewer built on the same model family as the author tends to share its
blind spots. If the harness lets you pick a different model/profile for
review agents than was used to write the change, prefer that.

**Match model cost to the aspect.** Cheap/fast checks (conventions,
formatting-adjacent comment review) don't need a frontier model; reserve
the most capable model for deep correctness and security analysis on
risky changes.

Give each agent: the diff (or file list), the files changed, and a short
description of what the change does (from the PR description, commit
messages, or the user's explanation).

### If an agent fails

1. Retry once — transient failures are common.
2. If the retry also fails, list the aspect under **Aspects skipped** in
   the summary with a one-line reason. Never drop an aspect silently.

## Step 5: Aggregate results

Produce one summary. Deduplicate: if two agents flag the same underlying
issue, keep the more specific finding and drop the other. Group findings
that share a root cause into a single entry with all affected locations,
rather than repeating the same finding per instance.

```markdown
# Review Summary

**Risk tier:** low | medium | high — one line on why

## Blocking (must fix)
- [agent]: finding [file:line]

## Suggestions (should fix)
- [agent]: finding [file:line]

## Nits (optional)
- [agent]: observation [file:line]

## Strengths
- What's well done in this change

## Aspects skipped
- [aspect]: reason

## Next steps
1. Address blocking issues first.
2. Re-run `/review` on the affected aspects only, once fixed.
```

### Avoid runaway review loops

If this skill is run repeatedly against the same branch/PR as fixes land:

- Cap it at **three review rounds** before stepping back and suggesting a
  different approach rather than continuing to iterate.
- After the first round, review only the new diff since the last review —
  don't re-litigate findings already resolved or already dismissed by the
  user.
- Never repeat a finding that was already raised and intentionally left
  unaddressed; note once that it's still open, not every round.

## Step 6: Posting a review (optional)

Only do this if explicitly asked to "post a review" or "submit feedback."

Build one batched review via the GitHub API rather than many separate
comments:

```bash
gh api repos/{owner}/{repo}/pulls/<number>/reviews \
  --method POST \
  --input REPLACE_ME.json
```

Payload shape:

```json
{
  "event": "REQUEST_CHANGES or COMMENT",
  "body": "Summary: what the change does, overall assessment, and any findings without a specific line (commit hygiene, missing version gating, etc.).",
  "comments": [
    {
      "path": "path/to/file.py",
      "line": 42,
      "side": "RIGHT",
      "body": "(blocking): explanation of the issue.\n\n```suggestion\nreplacement code\n```"
    }
  ]
}
```

- Prefix each inline comment with its severity so it's scannable:
  `(blocking)`, `(suggestion)`, or `(nit)`.
- Use the `suggestion` fenced-code block only for small, local fixes —
  GitHub renders it as an applyable diff. Don't use it for structural
  feedback.
- `event` is `REQUEST_CHANGES` if there's any blocking or suggestion
  finding, `COMMENT` if only nits (or a clean review with minor notes).
  Never select `APPROVE` unless the user explicitly asks for it — if the
  review is clean and would otherwise merit approval, ask first.

Always show the user the summary, the inline comments, and the chosen
`event` before posting, and wait for confirmation.

## Continuous improvement (optional)

If the project maintains a review knowledge base (a shared "what matters
here" doc per area, or a dataset of past PRs with known bugs), consult it
before reviewing and treat genuinely new, confirmed-correct findings as
candidates to feed back into it — this is what separates one-off reviews
from a review process that gets better over time.
