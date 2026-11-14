# Code review agents

Scaffolded subagents for use with Claude Code's `Agent`/`Task` tool. Each file
here defines one specialized reviewer, following the pattern: narrow scope,
explicit "what to look for," a confidence/severity bar to cut down noise, and
a consistent output format so results can be merged across agents.

| Agent | Focus | Typically applies when |
| --- | --- | --- |
| [`correctness-reviewer`](correctness-reviewer.md) | Logic errors, state handling, failure modes | Always, for any non-trivial change — run first/foreground |
| [`security-reviewer`](security-reviewer.md) | Exploitable vulnerabilities, auth, secrets | Change touches input handling, auth, data access, or config |
| [`architecture-reviewer`](architecture-reviewer.md) | Design fit, coupling, abstraction | New modules, public interfaces, or cross-cutting changes |
| [`code-quality-reviewer`](code-quality-reviewer.md) | Conventions, comment accuracy, simplicity | Always — cheap, high-signal pass |
| [`performance-reviewer`](performance-reviewer.md) | Algorithmic/I-O regressions at scale | Hot paths, data access code, loops over external data |
| [`test-reviewer`](test-reviewer.md) | Test coverage and quality | Test files changed, or new behavior added without tests |

## Using these scaffolds

1. **Fill in the gaps.** These are intentionally generic — adjust the
   confidence thresholds, add stack-specific checks (e.g. your framework's
   idiomatic error handling), and point `code-quality-reviewer` at real files
   in `.claude/rules/` once you have them.
2. **Dispatch in parallel.** Each agent only needs a diff, the list of
   changed files, and a short description of the change. Run them
   concurrently and run `correctness-reviewer` in the foreground since it
   typically needs to read the most surrounding context.
3. **Not every agent applies to every change.** A docs-only PR doesn't need
   `performance-reviewer`; a config-only change may skip `test-reviewer`.
   Decide per-diff which agents are relevant.
4. **Aggregate before reporting.** Merge findings into one summary grouped
   by severity (blocking / suggestion / nit), deduplicate overlapping
   findings across agents, and explicitly list any aspect that was skipped
   and why. This mirrors the risk-tiered, multi-agent review pattern where a
   lightweight pass can auto-approve low-risk changes and only escalate to
   deeper, specialized review as risk increases.
5. **Keep authoring and reviewing separate.** Where possible, review with a
   different model/profile than the one that authored the change — same-model
   review tends to share the same blind spots.

Pair this folder with a `/review`-style skill (in `.claude/skills/`) that
owns the orchestration logic: identifying the diff, deciding which agents
apply, dispatching them, and aggregating the results. Keeping that
dispatch/aggregation logic in a skill rather than duplicating it in each
agent keeps every agent here focused and reusable on its own.
