# Repository Guide

<!--
SCAFFOLD NOTES (delete this comment block once filled in):
- This file is always-on context: it loads at the start of every session and
  is re-injected on compaction, so keep it under ~200 lines and keep every
  line true and current.
- Put only what Claude needs in *every* session here. If a rule only applies
  to one subdirectory, put it in a nested `<dir>/CLAUDE.md` instead (it loads
  on-demand when Claude reads files in that directory).
- If a rule is cross-cutting but only touches *some* files (e.g. all API
  handlers, all migrations), use a path-scoped file in `.claude/rules/` with
  a `paths:` frontmatter filter instead of duplicating it here.
- If something is a multi-step procedure (deploy, release, review checklist),
  put it in `.claude/skills/` so it only loads when invoked, not every turn.
- If something must *always* or *never* happen deterministically (run a
  linter, block `rm -rf`, post to Slack), use a hook in `.claude/` settings,
  not a prose instruction here — prose is a request, not a guarantee.
- Replace every `REPLACE_ME` below. Delete sections that don't apply instead
  of leaving them empty.
-->

## What this project is

REPLACE_ME: one or two sentences on the product, service, or package, its
architecture style, and its primary stack.

## Commands

- Install dependencies: `REPLACE_ME`
- Run locally / dev server: `REPLACE_ME`
- Run tests: `REPLACE_ME`
- Run a single/focused test: `REPLACE_ME`
- Run type checks: `REPLACE_ME`
- Run lint / format: `REPLACE_ME`
- Build: `REPLACE_ME`
- Regenerate generated files (if any): `REPLACE_ME`

## Project layout

- `REPLACE_ME/` contains ...
- `REPLACE_ME/` contains ...
- `REPLACE_ME/` contains ...
- Tests live `REPLACE_ME` (e.g. next to source as `*.test.ts`, or under `tests/`).

> For deeper detail on a specific area, link out instead of inlining it, e.g.
> "for the payment processor, see `docs/payment-processor.md`." Large repos
> should treat this root file as a router to nested docs, not an encyclopedia.

## Conventions

- `REPLACE_ME`: Ideally link a `docs/engineering-standards/` folder that contains your standards for the project.
- Match existing patterns before introducing new ones.
- Keep changes scoped to the requested task; avoid unrelated refactors.
- Prefer existing utilities and dependencies over adding new ones.
- `REPLACE_ME`: language/stack-specific norms that aren't obvious from reading
  the code (e.g. "Python 3.x only", "Vite, not Create React App", naming
  conventions, error-handling patterns).

## Testing rules

- Run `REPLACE_ME` for focused changes (single file/package).
- Run `REPLACE_ME` (full suite) before completing broad refactors.
- Add or update a test for every behavior change or bug fix when practical.

## Generated files

- REPLACE_ME: list files/directories that are generated, where the real
  source of truth lives, and the exact command to regenerate them. Do not
  hand-edit generated output.

## Security & safety

- Do not commit secrets, credentials, or API keys.
- Do not run destructive commands (e.g. force-push, drop database, `rm -rf`)
  without explicit approval.
- REPLACE_ME: production boundaries, approval rules, or migration-safety
  requirements specific to this project.

## Verification before finishing work

Run the relevant focused checks, then the required project-wide checks:

1. `REPLACE_ME`
2. `REPLACE_ME`
3. `REPLACE_ME`

If a command fails, fix the issue and rerun it rather than reporting work as
done.

## PR / handoff expectations

- REPLACE_ME: required evidence (test output, screenshots), changelog notes,
  or review checks expected before a change is considered mergeable.
