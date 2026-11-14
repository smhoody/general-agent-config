# Agent-config

A personal, portable collection of scaffolds for **agentic engineering**:
customizing AI coding agents (Claude Code, Copilot, etc.) and making a
project **agent-ready** so those agents can work reliably in it.

Nothing in here is tied to a specific project. Each file is a generic
starting point — copy what's relevant into a target repository, fill in
the `REPLACE_ME` placeholders, and delete what doesn't apply.

All work is based on the materials from the Stanford course [CS146S: The Modern Software Developer](https://themodernsoftware.dev/).

## Why this exists

Agents fail for predictable reasons: no tests to verify correctness, no
docs to stay consistent with, an unreliable build, or no playbook telling
them how the project works. The fix isn't a smarter agent — it's a
better-prepared repository and a deliberate set of customizations:
always-on context, hard rules, reusable procedures, and specialized
reviewers. This repo collects working scaffolds for all of that so the
pattern doesn't have to be reinvented per project.

See [docs/notes.md](docs/notes.md) for the full set of source notes this
repo is built from — context engineering, agent skills, customizing
agentic setups, agent-ready repositories, and agentic code review.

## What's in here

```
CLAUDE.md                          # Root always-on context scaffold
.claude/
  agents/                          # Specialized review subagents
    correctness-reviewer.md
    security-reviewer.md
    architecture-reviewer.md
    code-quality-reviewer.md
    performance-reviewer.md
    test-reviewer.md
    README.md                      # How the review agents compose
  rules/
    commenting-standards.md        # Hard, path-scoped commenting rules (Python example)
  skills/
    review.md                      # Orchestrates the review agents end to end
    readiness-report.md            # Assesses and improves agent-readiness
docs/
  notes.md                         # Source notes this repo distills
```

- **[CLAUDE.md](CLAUDE.md)** — the root, always-loaded project context
  file: overview, commands, layout, conventions, verification steps, and
  safety rules. Keep it under ~200 lines; add nested `<dir>/CLAUDE.md`
  files for subsystem-specific context instead of growing this one.
- **[.claude/agents/](.claude/agents)** — subagents for delegated,
  isolated work. The six scaffolded here are specialized code reviewers;
  see that folder's own README for how they're meant to be dispatched and
  aggregated.
- **[.claude/rules/](.claude/rules)** — hard constraints, loaded either
  always (no `paths:` frontmatter) or only for matching files (with
  `paths:`). Use rules for things that must always/never happen in a
  specific kind of file, rather than repeating them in every relevant
  CLAUDE.md.
- **[.claude/skills/](.claude/skills)** — reusable, multi-step procedures.
  Only a skill's name/description load at session start; the full body
  loads when invoked. Use skills for anything that's a process (a review
  workflow, a release checklist, a deploy flow) rather than static
  context.
- **[docs/notes.md](docs/notes.md)** — the underlying research/notes this
  whole repo is distilled from. Treat it as the rationale; treat the
  scaffolds as the implementation.

## Making a project agent-ready

A quick checklist, drawn from `docs/notes.md`, for bringing a project up
to a level where agents can work in it with minimal supervision:

1. **Sufficient test coverage** — without it, nothing can verify
   correctness.
2. **A hermetic, reliable build** — without it, nothing can verify a
   change compiles/runs.
3. **Consistent, documented patterns** — no stale READMEs; conventions an
   agent can actually infer or be told.
4. **Fast, cheap, automated checks** — lint, format, and tests that run on
   every commit, not just in CI five minutes later.
5. **An agent playbook** — a root `CLAUDE.md`, plus rules/skills/subagents
   for anything that needs to be deterministic, reusable, or delegated.

In roughly this order: write a `CLAUDE.md` that explains how to build,
test, and run the project in plain language; make sure one command runs
the tests; add lint/format on commit; add a `.env.example` so nobody has
to guess required configuration; and prefer writing a failing test first
so an agent can fix it against a clear target.

## Using this repo

1. Start from [CLAUDE.md](CLAUDE.md) — fill it in for the target project
   first; everything else builds on having accurate project context.
2. Pull in `.claude/agents/`, `.claude/rules/`, and `.claude/skills/` as
   needed — they're independent, so take only what's relevant (e.g., the
   review pipeline doesn't require every rule file to exist first).
3. Keep this repo itself up to date as a personal library: when a
   correction or a new pattern proves useful in a real project, port it
   back here so the next project starts from the improved version.
