# Agentic Engineering

## Context Engineering Guide

Objective: adopt a reliabe synchronous AI development workflow and use AI to rapidly onboard to unfamiliar codebases

RePPIT framework:
- Research
- Propose
- Plan
- Implement
- Test

Best practices:
-  review specs VERY carefully
- For large features, break into task lists
    - When an item is finished, document agent work in a written artifact, which becomes context for the next item
- Be mindful of context window (1/2 - 3/4 of "smart" tokens)

Task delegation taxonomy:
- Simple prompting: copy changes, small fixes
- Make a plan: small features across 3-5 files
- Research then plan: medium features across multiple project and repos
- (Research then plan) x N: big refactors, whole new features

Save design tradeoffs/decision points for parts of codebase in a .notes/ directory (can be local or shared)

Context management (as of 2026/11/13):
- Optimum context level to compact is at 314k tokens (or between 250k-450k token).
- Before 146k tokens, you use more tokens compacting than you would from continuing in the context.
- Each doubling of context costs roughly 6x more, so be deliberate about which work earns a big context window. 
- There is almost never a good reason to let an agent run near 1M tokens.



## Agent Skills

Best practices:
- Keep scope focused: one skill, one clearly defined task
- Maintain your own collection of skills that you can easily port across projects
- Check your chat transcripts: skills are processes written as markdown - look at your own chats to see where you intervened or corrected
- Don't burn tokens/time reinventing the wheel: equip agents with a CLI tool and wrap the boilerplate in a skill
	- E.g., writing a script to read firestore DB
- Include situations and request phrases that should trigger the skill
- Progressive dislosure: reference specific files as need with additional context, modularize and compose skills
- Maintain a verification skill to have an agent check its work


## Customizing Agentic Setup

Techniques for customizing agents:
- Agent behavior files (CLAUDE.md, AGENTS.md)
- Hooks
- Skills
- Subagents

Designing an optimal AGENTS.md
- Structure:
	- Project overview (map of codebase)
	- Build and test commands
	- Code style guidelines
	- Testing instructions
	- Security considerations
- Language or stack-specific notes: Python 3.14 only, Vite over React, etc.
- Make instructions as universally applicable as possible
- Avoid code snippets, they get outdated
- Be mindful of length (<200 lines)
- Large repos should use nested config files with the rool AGENTS.md as a router
    - e.g.: for more details about Payment Processor, refer to docs/payment-processor.md

Agents/claude.md example:
```markdown
# Repository guide
 
## Commands
 
- Install dependencies: `pnpm install`
- Start local server: `pnpm dev`
- Run unit tests: `pnpm test`
- Run type checks: `pnpm typecheck`
- Run lint: `pnpm lint`
 
## Project layout
 
- `app/` contains routes and server-rendered pages.
- `components/` contains reusable UI components.
- `lib/` contains shared business logic and integrations.
- Tests live next to the code they cover as `*.test.ts` or `*.test.tsx`.
 
## Coding conventions
 
- Prefer small, focused modules with explicit types.
- Keep business logic in pure functions where possible.
- Match existing component and file naming patterns.
- Do not add a new dependency unless the existing stack cannot solve the problem.
 
## Verification
 
Before calling work finished, run:
 
1. `pnpm test`
2. `pnpm typecheck`
3. `pnpm lint`
 
If a command fails, fix the issue and rerun it.
```

Agent.md steps:
1. Create one root file: start with one file, then add nested files only where a package, app, or service needs different rules
2. Put commands first: list exact install, development, test, type check, lint, and build commands
3. Add convetions and safety fules: capture repo layout, generated-fule rules, security boundaries, ownership boundaries, and required proof before agent work is completed
4. Commit with the project: project instructions belong in git


| Section | What good looks like |
| --- | --- |
| Project overview | One short paragraph explaining the product, architecture, or package. |
| Commands |	Exact install, local run, test, lint, type check, build, and regeneration commands. |
| Repository map |	Directories Droid must understand before editing. |
| Conventions |	Project-specific patterns that are not obvious from code. |
| Testing rules |	Focused test commands, full gate commands, and when to add tests. |
| Generated files |	Which files are generated, what source to edit, and how to regenerate. |
| Security rules |	Secret handling, production boundaries, approval rules, and migration safety. |
| PR expectations |	Required evidence, screenshots, changelog notes, or review checks. |

Prefer concrete rules:
```markdown
## Testing
- Run `pnpm test -- path/to/file.test.ts` for focused changes.
- Run `pnpm test` before completing broad refactors.
- Add a regression test for every bug fix when practical.
```

Root template:
```markdown
# Repository guide
 
## What this project is
 
One or two sentences describing the product, service, or package.
 
## Commands
 
- Install: `REPLACE_ME`
- Run locally: `REPLACE_ME`
- Run tests: `REPLACE_ME`
- Run type checks: `REPLACE_ME`
- Run lint: `REPLACE_ME`
- Build: `REPLACE_ME`
 
## Project layout
 
- `REPLACE_ME/` contains ...
- `REPLACE_ME/` contains ...
- `REPLACE_ME/` contains ...
 
## Conventions
 
- Match existing patterns before introducing new ones.
- Keep changes scoped to the requested task.
- Prefer existing utilities and dependencies.
- Add or update tests when changing behavior.
 
## Verification
 
Before completing work, run the relevant focused checks and then the required project checks:
 
1. `REPLACE_ME`
2. `REPLACE_ME`
3. `REPLACE_ME`
 
## Safety
 
- Do not commit secrets or credentials.
- Do not run destructive commands without explicit approval.
- Do not modify generated files by hand. Regenerate them with `REPLACE_ME`.
```

Seven ways to customize Claude: 
- CLAUDE.md: for always-on project context
	- Located at the root, it loads into context at session start and stays.
	- Includes: build commands, directory layout, monorepo structure, coding conventions, and norms
	- Two types:
		- Always loaded: root md file that always gets loaded into the session
		- On-demand: md files in subdirectories below the folder where you initialized the session
			- Example: app/api/CLAUDE.md loads when Claude reads a file in app/api/, rather than session start.
	- Since it's loaded into context every time:
		- Keep CLAUDE.md under 200 lines
		- Make sure the content is relevant for the directory
			- e.g. root-level CLAUDE.md is broad and describes project overview, but src/processors/CLAUDE.md specifically describes processor structure and standards.
- Rules: hard constraints saved in .claude/rules/
	- Unscoped rules behave like CLAUDE.md where they always get loaded at session start and get re-injected on compaction
	- Path-scoped rules allow you to load rule instructions only when they are relevant by adding a 'paths' field that controls when they load.
		- E.g. a rule scoped to src/api/** stays out of context during a docs-only session, and would only be loaded when Claude reads files within src/api/. 
		- Rule content example:
            ```markdown
			---
			paths:
			  - "src/api/**"
			  - "**/*.handler.ts"
			---
			All API handlers must validate input with Zod before processing.
            ```
			- Tip: Reach for a path scoped rule over a nested CLAUDE.md file when the instruction regards a cross-cutting concern or file that appears in multiple (but not all) corners of the codebase.
				- e.g.: if there are several migration folders in your repo, create a .claude/rules/migration.md rule rather than adding your constraint to a CLAUDE.md file in each directory.
- Skills: reusable procedures that live in .claude/skills/
	- Only the name and description load at session start; the fully body loads when Claude invokes the skill (through slash command or auto-matching)
	- Tip: instructions that are procedural, like deploy workflows, release checklists, or review processes, belong in a skill rather than in CLAUDE.md. 
- Subagents: delegated work defined in .claude/agents/
	- Isolated assistants for specific side tasks
	- Similar to skills in that the name, description, and tool list load at session start
	- Called by Claude via Agent tool with a prompt string
	- Subagents can nest up to 5 levels deep, and dynamic workflows orchestrate tens to hundreds of background agents without requiring specific details for each subagent architecture
	- Intermediate results live in script variables rather than in Claude's context window
	- Tip: Use a subagent when a side task like deep search, log analysis pass, or a dependency audit would cloutter your main conversation with inermediate results you won't reference. Use a skill when you want the procedure to play out inside the main thread so you can see and steer each step.
- Hooks: deterministic automation
	- Registered in settings.json, managed policy settings, or skill/agent frontmatter
	- Provide more deterministic control over Claude's behavior by firing on specific events in Claude's lifecycle like file edits, tool calls, or session start. 
	- Types: command, HTTP, mcp_tool, prompt, and agent. The first three execute deterministically, latter two use Claude's judgment rather  than a set of rules
	- Low context cost because the configuration lives outside the main context window. The harness runs the handler (deterministic hook types) or makes model calls with separate windows.
	- Tip: use hooks for anything that should happen deterministically: running liters, posting to Slack on completion, blocking specific commands (rm rf) before the execute. 
- Output stypes/System-prompt appends: global changes
	- Files in .claude/output-styles/ that inject instructions into the system prompt.
	- Like sys prompts, they always load at the start of every session and are never compacted. They get cached after the first request within a session so it has moderate context cost.
	- Changes the output style and will replace the default output style (unless you set keep-coding-instructions: true in the style's frontmatter).
	- Changing the output style would override the instructions that tell Claude it is helping users with software engineering tasts.
	- Probably don’t change this
	
Good method usage, rather than placing in a CLAUDE.md:
- "Every time X, always do Y": use a hook in settings.json
- "Never do this": hook. Guardrails should be deterministic - if they're not, Claude is not guaranteed to follow a rule.
- 30-line procedure: use skills
- API-specific rule wihtout paths: if a rule only applies to src/api/**, scope it with 'paths:' to keep it out of context during unrelated work. 
- Writing personal preferences to a project-level CLAUDE.md: use local files for personal preferences, keep project-level files for preferences that are team-wide. 


## Agent-ready Repositories

Much of LLM confusion comes from trying to finish a task with a messy or incomplete developer environment.

People can work around insufficient test coverage, outdated documentation, no linter, flakey build, complex setup, etc. But this breaks LLMS. 
No tests = can't verify correctness
No docs/standards = can't ensure consistency
Unreliable build = can't verify changes
No observability = can't debug
Complex setup = no environment reproducibility
No security check = introduces vulnerabilities

Agent-ready environment checklist:
1. Sufficient test coverage
2. Hermetic build system 
3. Well-documented, consistent design patterns; no outdated READMEs
4. Verifiable checks
5. Agent-specific playbook: CLAUDE.md, rules, skills, hooks

Agent Isolation:
- Git worktrees: git worktree add -b new-branch ../new-branch
- Multi-agent orchestrators:
    - Claude Code Swarms, Conductor, Cursor parallel agents, or build your own
    - Abstract awal lower level setup like creaing worktrees
    - Cons: shared resources are not completed isolated (dbs, ports)

When agents go wrong, it is more likely than not a problem with the environment. 

Small, checked steps beats one end-check
Order: Pre-commit hooks, then local tests, then CI pipeline, then code review
Pre-commit hooks catch problems that CI takes 5 min to check

Five levels of readiness:
1. Functional: code runs
2. Documented: written
3. Standardized: enforced
4. Optimized: measured
5. Autonomous: hands-off

Level 1: code runs, but checking is manual
Level 3: checks are automated and enforced, but metrics aren't measured
Level 4+: fast feedback, ready for long solo runs.

Make a project agent-ready:
1. Write AGENTS.md: how to build, test, and run it, in plain words
2. One command runs the tests: put it in AGENTS.md
3. Lint and format on commit
4. Add .env.example: no guessing which variables exist
5. Write the failing test first: then let the agent fix the bug
6. Run /readiness-report: check level and make adjustments

Overall: Make checks cheap, fast, and small


## Agentic Code Review

Code review hierarchy of needs (least to most important):
1. Style
2. Find bugs
3. Design discussion
4. Correct solution
5. Mental alignment

AI-Native Code Review
Maturity ladder:
1. Human reviews everything: impossible to scale
2. Single-agent review: better, especially if combined with auto-fix loop but still leaves gains on the table
3. Multi-agent risk-tiered review with self-improvement built in
    1. Engineering code standards stored in skill knowledge base
    2. Best practice that many companies are converging on

Six layers of checks, with rollback under them:
1. Verification
2. Required CI
3. AI review
4. Security review
5. Human approval
6. Merge queue + flags

PR Author: owns the PR lifecycle end-to-end, iterates on fixes until addressed and merged.
PR Risk Analyzer: Routes every PR based on risk
Auto-approve: automatically approves low-risk PRs
Deep Code Review: Analyzes code correctness line by line
Review Outcome: Posts comments, approvals, or change requests
Code Review Memory: Org/Project-level memory storing agent akills and developer best practices

Flow: Start with PR Author, then PR Risk Analyzer, which utilizes the Auto-Approve and Deep Code Review checks. Deep Code Review moves to Review Outcome and uses Code Review Memory, then finally goes back to PR Author.

Pull Request -> Security Review, Architecture Review, Code Quality Review, Performance Review -> Aggragate Judge

Example for review guide:
```markdown
## What to Flag
- Inject vulnerabilities (SQL, XSS, command, path traversal)
- Authentication/authorization bypasses in changed code
- Hardcoded secrets, credentials, or API keys
- Insecure cryptographic usage
- Missing input validation on untrusted data at trust boundaries
## What NOT to flag
- Theoretical risks that require unlikely preconditions
- Defense-in-depth suggestions when primary defenses are adequate
- Issues in unchanged code that this MR doesn't affect
- "Consider using library X" style suggestions
```

Design of an agentic code reviewer
- Diversify model used for review compared to generation
- Use thoughtful context: (Docs + issue tickets, Codebase + codebase indexer) -> Review Agent
- Verify: spin up dev container to test different code flows
- Cost efficiency: pick model based on task complexity
    - Code context: cheap/fast model
    - Browser use: smaller/medium model, higher reasoning
    - Deep bug analysis: frontier model
- Evaluations
    - Historical PRs with known bugs
    - Detected incidents with reverts

Agent Review options:
- General Purpose: Claude Code, Greptile, Coderabbit, Devin
- Security: Vercel deepsec, Wiz security, Semgrep


Layered Code Review and Risk-based Human Oversight by Claude Team: https://x.com/addyosmani/status/2114604912520921136
- A New Definition of Review: When code volume surges, manual review becomes a major bottleneck. Anthropic shifted the core question from "Did a human read this code?" to"What could go wrong, and which automated or human check should catch it?" 
- The Multi-Layered Review Architecture:
    ○ Pre-PR Verification: Before opening a PR, Claude runs tests, builds, and linters, providing concrete evidence of execution, followed by a fresh-context subagentreview. 
    ○ Automated AI PR Review: Every PR undergoes an AI review where multiple agents hunt for bugs in parallel; a separate consensus group votes on findings to filter outstyle nits and surface actionable issues. 
    ○ Security & CI Enforcement: Mandatory blocking for security vulnerabilities, type checks, and comprehensive test suites. 
    ○ Risk-Scaled Human Approvals: Human approval requirements scale dynamically with change risk. Crucially, a bot's approval never counts as a human approval—the personwho prompted the AI is designated as the reviewer and owns the outcome ("You land it, you own it"). 
- Curbing Runaway AI Review Loops: Early on, agent authors and reviewers occasionally entered endless feedback cycles (e.g., one PR ran through over 50 review roundsovernight). To fix this, Anthropic introduced: 
    ○ A strict cap of three review rounds before forcing a strategic rethink. 
    ○ Restricting re-reviews to new diffs only after the first few passes. 
    ○ Banning repeated findings and grouping bug classes so agents fix all instances at once. 
- Where Humans Still Add Value: Instead of burning human time on routine syntax correctness, engineers focus on product direction, high-risk infrastructure (such as billing or security workflows), core architecture, and evaluating test evidence or UI screenshots. 
- Example config:
    CLAUDE.md -- The project description file that you can use to give Claude context about the project (technology stack, conventions, goals, non-goals). Claude Code pulls this file in for every single review, so we keep it brief for large codebases and reference the rest of the documentation here. When code breaks a convention in this file, or an update makes something in it out of date, the reviewer flags it.
    REVIEW.md -- The file where code owners tell the reviewer what matters in their part of the code. We separately verify whether the reviewer is actually following these guidelines. Good ones are brief and focus on:
    - Defining project failure (the docs should not lie; the config should not misroute) 
    - Limiting nitpicking 
    - Defining ignored paths 
    - Requiring file + line reference, not guessing filenames 
    - Requiring top-level summary including count and clear statement of "no issues"


Coordinating Specialist Review Agents at Scale: https://blog.cloudflare.com/ai-code-review
- The Problem with Code Reviews: Traditional code reviews are a major bottleneck, often sitting in queues for hours while reviewers context-switch to handle minor nitpicks. Early experiments with out-of-the-box AI tools lacked the required flexibility, while naive prompt approaches (shoving a git diff directly into an LLM) resulted in extreme noise, hallucinated syntax errors, and generic advice.
- The Multi-Agent Approach: Rather than relying on a single monolithic prompt, Cloudflare built a CI-native system around OpenCode (an open-source coding agent). When a merge request is opened, it is evaluated by up to seven specialized reviewers:
    - Security
    - Performance
    - Code Quality
    - Documentation
    - Release Management
    - Internal Engineering Codex Compliance
- The Coordinator Agent: A central coordinator agent oversees the specialists, deduplicates overlapping findings, evaluates the true severity of issues, and condenses everything into a single, clean, structured review comment.
- Impact at Scale: The system has processed tens of thousands of internal merge requests. It automatically approves clean code, flags real bugs with high accuracy, and actively blocks merges when severe vulnerabilities or critical problems are detected.
- Composable Plugin Architecture: To avoid tight coupling and future rewrites, the entire system is built using a modular plugin architecture (ReviewPlugin interface). This allows Cloudflare to seamlessly swap out version control systems (such as GitLab), AI providers, and compliance standards independently.


How to build a Code Review Agent: https://www.greptile.com/blog/how-to-build-a-code-review-agent
- The Basic Code Review Workflow
    - A basic review agent can be set up in a day using a sandbox connected to GitHub webhooks via a model like Codex or Claude Code. However, out-of-the-box basic setups typically suffer from several issues:
    - High hallucination and blind spots: Missing complex bugs or lacking deep repository context.
    - Prohibitive cost: Running large frontier models on every commit costs $2–$20 per run.
    - Noise: Generating annoying, nitpicky comments or failing to catch chained logic bugs.
- 5 Key Improvements for Production-Grade Agents
    1. Prompt Optimization & Historical Evaluation
        - Prompting: Instruct the agent specifically on what bug categories to focus on and explicitly forbid stylistic nitpicks.
        - Evaluation Datasets: Because true bugs are rare in normal workflows, build a dedicated benchmark dataset of historical PRs containing known bugs to measure performance iteration.
    2. Model Inversion (Cross-Model Review)
        - Blind Spot Mitigation: Authors and reviewers using the same model family share identical blind spots (e.g., Claude models are twice as likely to introduce and miss N+1 SQL errors compared to GPT models).
        - Cross-Reviewing: Use a router or classifier to ensure the review model differs from the author model (e.g., use GPT models to review Claude-authored code), increasing bug catch rates by roughly 10%.
    3. Deep Codebase Context & Wikis
        - Context Access: Integrate an MCP (Model Context Protocol) gateway to connect the agent to documentation, Jira, or Linear.
        - Codebase Indexing: Maintain an automated, periodically crawled repository wiki—including logs of past reverts and production incidents—so the agent understands historical patterns, product intent, and chained dependencies (like complex auth flows).
    4. Autonomous Runtime Testing
        - Active Verification: Instead of solely relying on static text analysis of a git diff, configure dev boxes to spin up PR branches, launch development servers, and use computer-use / browser automation to pressure-test user flows and UI behavior.
    5. Cost Allocation (Task-Specific Model Routing)
        - Tiered Processing: Avoid running expensive frontier models on every token. Use cheap, lightweight models to filter out irrelevant file reads and perform initial reconnaissance, passing only high-signal code contexts to frontier reasoning models.



### Credit
- https://themodernsoftware.dev/
- https://claude.com/resources/articles/steering-claude-code-skills-hooks-rules-subagents-and-more
- https://x.com/walden_yan/status/2047054401341370639
- https://langwatch.ai/blog/context-tax-when-to-compact
