---
name: readiness-report
description: >
  Assess whether a project is agent-ready and make adjustments to raise its
  readiness level. Use when asked to "run a readiness report", "make this
  project agent-ready", "prepare this repo for agents", "onboard this repo
  for AI coding agents", or to audit why agents keep struggling in a repo.
---

# Agent-Readiness Report

<!--
SCAFFOLD NOTES (delete once adapted):
- Most agent failures trace back to the environment, not the model: no
  tests means no way to verify correctness, no docs means no way to stay
  consistent, an unreliable build means no way to verify a change, and a
  missing playbook means the agent has to guess conventions every time.
  This skill exists to find and close those gaps before they cause a
  failed task, rather than diagnosing them after the fact.
- This is an assess-then-fix skill: always produce the report first, then
  propose concrete changes, and only apply changes the user confirms.
- Add project- or org-specific checks to Step 1 as you find them (e.g. a
  required security scan, a specific CI provider's config format).
-->

## Step 1: Assess the five pillars

Investigate the repository directly — read config files, CI definitions,
and existing docs rather than asking the user to describe the setup.

1. **Test coverage** — Is there a test suite? Can you find and run it with
   a single command? Does it cover more than a trivial fraction of the
   code (check for a coverage tool/report if one exists)? Are critical
   paths and error handling covered, not just happy paths?
2. **Hermetic, reliable build** — Does installing dependencies and
   building/running the project work the same way every time, on a clean
   checkout? Are versions pinned (lockfile, pinned base image, pinned
   toolchain version)? Is there a `.env.example` (or equivalent) so
   required configuration doesn't have to be guessed?
3. **Documented, consistent patterns** — Is there a root `CLAUDE.md` or
   `AGENTS.md`? Does it (or the README) accurately describe how to build,
   test, run, and lint the project — not just what the project is? Do a
   handful of spot-checks: does the documented test command actually
   work? Does the directory layout it describes match reality? Flag
   anything stale.
4. **Fast, automated, enforced checks** — Is there lint/format/type
   checking configured? Does it run automatically (pre-commit hook, CI),
   or only when someone remembers to? How long does the fastest layer of
   checks take — seconds, or minutes? Checks that are slow or manual tend
   to get skipped, which defeats the point.
5. **Agent-specific playbook** — Beyond a root context file: are there
   `.claude/rules/` for hard constraints that must always/never happen
   (e.g. "never hand-edit generated files")? Are there `.claude/skills/`
   for multi-step procedures the project repeats (release, review,
   migration)? Are there `.claude/agents/` for specialized/delegated work
   (e.g. code review)? Do hooks exist for anything that must be
   deterministic (blocking destructive commands, running a linter on every
   edit)?

For each pillar, note: what exists, what's missing, and what's actively
wrong (e.g., a README that documents a command that no longer works) —
wrong information is worse than no information, since it actively misleads
the agent instead of leaving a gap to investigate.

## Step 2: Map findings to a readiness level

Use the five findings to place the project on this ladder. A project is at
a level only once every item in it (and below) holds — don't round up.

1. **Functional** — the code runs, but verifying a change requires a human
   to manually build/run/click through it.
2. **Documented** — build/test/run/lint instructions are written down and
   accurate (a `CLAUDE.md`/`AGENTS.md` or equivalent), so an agent doesn't
   have to reverse-engineer them from source.
3. **Standardized** — the checks from Step 1.4 are automated and enforced
   (pre-commit and/or required CI), not just documented as something a
   human should remember to run.
4. **Optimized** — the same checks are fast and their health is measured
   (coverage trend, flake rate, check duration) so regressions in the
   checks themselves are visible, not just regressions in the product.
5. **Autonomous** — feedback is fast and reliable enough that an agent can
   run a long, mostly-unsupervised session and self-correct from the
   checks alone, without a human in the loop at every step.

State the current level plainly, and name the single pillar gap that's
blocking the next level — don't list every possible improvement as equally
urgent.

## Step 3: Propose adjustments

Turn the gaps into a short, ordered action list. In rough priority order
(earlier items unblock later ones and are usually cheaper):

1. **Write or fix the playbook first.** If `CLAUDE.md`/`AGENTS.md` is
   missing, stale, or wrong, fix that before anything else — every other
   improvement is less useful if the agent can't discover it. Keep it
   short, plain-language, and accurate; link out to deeper docs instead of
   inlining them.
2. **Guarantee one command runs the tests**, and that command is the one
   written down in the playbook. If running tests requires multiple
   manual steps today, wrap them in a single script/task first.
3. **Add lint/format (and type-check, if applicable) as a pre-commit
   hook**, not just a CI step — pre-commit catches a problem in seconds
   that CI would take minutes to report back.
4. **Add a `.env.example`** (or equivalent) enumerating every required
   environment variable, so environment setup isn't trial-and-error.
5. **Favor a failing-test-first workflow for bug fixes**: once a bug is
   reported, write a test that reproduces it (red) before handing the fix
   to an agent, so success is a concrete, checkable target (green) rather
   than a subjective judgment call.
6. **Re-run this assessment after changes land** to confirm the level
   actually moved, rather than assuming it did.

Favor making checks **cheap, fast, and small** over adding more of them —
a single slow end-to-end check that only runs in CI is far less useful
than several small checks that run on every save or commit.

## Step 4: Apply changes (with confirmation)

Show the user the report (current level, pillar findings, ordered action
list) before changing anything. For each action they approve:

- Reuse what already exists in this project's own scaffolds where
  possible — e.g. start from this repo's root `CLAUDE.md` template rather
  than writing a playbook from scratch, and from `.claude/rules/` /
  `.claude/skills/` patterns already established here.
- Make changes incrementally and verify each one (e.g., actually run the
  new pre-commit hook, actually run the documented test command) rather
  than writing configuration you haven't confirmed works.
- Don't silently skip an approved action — if something blocks it (missing
  permissions, a decision only the user can make), say so explicitly.

## Optional: isolating concurrent agent work

If the project expects multiple agents or sessions to work on it at once,
note this as a recommendation rather than a blocking gap:

- Use git worktrees to give each agent/session an isolated working copy
  on its own branch: `git worktree add -b <branch> <path>`.
- Be aware that worktrees don't isolate shared resources — a local
  database, dev server port, or cache can still collide between
  concurrent sessions. Flag this if the project's dev setup has such
  shared state and concurrent agent use is expected.

## Output format

```markdown
# Agent-Readiness Report

**Current level:** <Functional|Documented|Standardized|Optimized|Autonomous>
**Blocking gap for next level:** <one sentence>

## Pillar findings
- Test coverage: <what exists / what's missing or wrong>
- Build: <...>
- Docs & consistency: <...>
- Automated checks: <...>
- Agent playbook: <...>

## Recommended actions (ordered)
1. ...
2. ...

## Changes applied this session
- <only if Step 4 was run>
```
