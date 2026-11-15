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

From [Claude Code Docs](https://code.claude.com/docs/en/skills):

### Frontmatter types
| Field | Description |
|---|---|
| name | Command name shown in the `/` menu |
| description | What the skill does and when to use this. Claude uses this to decide when to apply the skill |
| when_to_use | Additional context for when Claude should invoke the skill, such as trigger phrases or example requests |
| arguments | Named positional arguments for `$name substitution` in the skill content. Accepts a space-separated string or a YAML list |
| disable-model-invocation | When set to `true`, it prevents Claude from automatically loading this skill |
| user-invocable | Set to `false` when only Claude should invoke the skill (Claude Code hides it from the `/` menu) |
| allowed-tools | Tools Claude can use without asking permission during the turn that involves this skill |
| disallowed-tools | Tools removed from Claude's available pool while this skill is active (e.g. `AskUserQuestion`) |
| model | Model to use for this skill. Override applies for the rest of the current turn and resumes to session model when complete |
| effort | Effort level for the rest of the turn. Options: `low`, `medium`, `high`, `xhigh`, `max` |
| context | Set to `fork` to run in a forked subagent context |
| background | Only applies with `context: fork`. Set to `false` to wait for the forked subagent's result in the turn that invoked the skill, instead of running it in the background. Default: `true` |
| paths | Glob patterns that limit when this skill is activated. When set, Claude loads the skill automatically only when working with files matching the patterns. |

Skill example:
```yml
---
name: deploy
description: Deploy the application to production
context: fork # runs the skill in its own subagent context
disable-model-invocation: true # prevents Claude from triggering this skill automatically (can only be run manually: /deploy)
allowed-tools: Bash(git add *) Bash(git commit *) Bash(git status *)
model: claude-opus-5-5
effort: medium
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: '"${CLAUDE_PROJECT_DIR}"/scripts/security-check.sh' # runs security validation script before each Bash cmd
---

Deploy the application:
1. Run the test suite
2. Build the application
3. Push to the deployment target
```

### Supporting files
```
my-skill/
├── SKILL.md (required - overview and navigation)
├── reference.md (detailed API docs - loaded when needed)
├── examples.md (usage examples - loaded when needed)
└── scripts/
    └── helper.py (utility script - executed, not loaded)
```
Reference supporting files from SKILL.md so Claude knows what each file contains and when to load it:
```md
## Additional resources

- For complete API details, see [reference.md](reference.md)
- For usage examples, see [examples.md](examples.md)
```


From [agentskills.io](https://agentskills.io/skill-creation/best-practices):
### Refine execution
The first draft of a skill usually needs refinement. Run the skill against real tasks, then feed the results — all of them, not just failures — back into the creation process. Ask: what triggered false positives? What was missed? What could be cut?

**Read agent execution traces**, not just final outputs. If the agent wastes time on unproductive steps, common causes include **instructions that are too vague** (the agent tries several approaches before finding one that works), instructions that don’t apply to the current task (the agent follows them anyway), or too many options presented without a clear default.

### Using context wisely
Deciding what a skill should cover is like deciding what a function should do: you want it to encapsulate a coherent unit of work that composes well with other skills. 
- Skills scoped too narrowly force multiple skills to load for a single task, risking overhead and conflicting instructions. 
- Skills scoped too broadly become hard to activate precisely.

Overly comprehensive skills can hurt more than they help — the agent struggles to extract what’s relevant and may pursue unproductive paths triggered by instructions that don’t apply to the current task. Concise, stepwise guidance with a working example tends to outperform exhaustive documentation. When you find yourself covering every edge case, consider whether most are better handled by the agent’s own judgment.

**Recommended to keep skills under __500 lines and 5,000 tokens__**

When a skill legitimately needs more content, move detailed reference material to separate files in `references/` or similar directories.

### Calibrating control
**Give the agent freedom** when multiple approaches are valid and the task tolerates variation. 

For flexible instructions, explaining why can be more effective than rigid directives — an agent that understands the purpose behind an instruction makes better context-dependent decisions. A code review skill can describe what to look for without prescribing exact steps:
```md
## Code review process

1. Check all database queries for SQL injection (use parameterized queries)
2. Verify authentication checks on every endpoint
3. Look for race conditions in concurrent code paths
4. Confirm error messages don't leak internal details
```

**Be prescriptive** when operations are fragile, consistency matters, or a specific sequence must be followed:
```md
## Database migration

Run exactly this sequence:

```bash
python scripts/migrate.py --verify --backup
```

Do not modify the command or add additional flags.
```

A skill should teach the agent *how to approach* a class of problems, not *what to produce* for a specific instance. Compare:
```md
<!--> Specific answer — only useful for this exact task -->
Join the `orders` table to `customers` on `customer_id`, filter where
`region = 'EMEA'`, and sum the `amount` column.

<!-- Reusable method — works for any analytical query -->
1. Read the schema from `references/schema.yaml` to find relevant tables
2. Join tables using the `_id` foreign key convention
3. Apply any filters from the user's request as WHERE clauses
4. Aggregate numeric columns as needed and format as a markdown table
```


### Patterns for effective instructions
- Gotcha sections: contain environment-specific facts that defy reasonable assumptions. 
    - Example:
    ```md
    ## Gotchas
    - The `users` table uses soft deletes. Queries must include
    `WHERE deleted_at IS NULL` or results will include deactivated accounts.
    - The user ID is `user_id` in the database, `uid` in the auth service,
    and `accountId` in the billing API. All three refer to the same value.
    - The `/health` endpoint returns 200 as long as the web server is running,
    even if the database connection is down. Use `/ready` to check full
    service health.
    ```
    - Keep gotchas in SKILL.md where the agent reads them before encountering the situation. A separate reference file works if you tell the agent when to load it, but for non-obvious issues, the agent may not recognize the trigger.

- Output templates: when you need an agent to produce output in a specific format.
    - Short templates can live inline in the `SKILL.md`, longer templates (or templates only needed in certain cases) should be stored in `assets/` and reference them from `SKILL.md` so they only load when needed.
    - Example:
    ```md
    ## Report structure
    Use this template, adapting sections as needed for the specific analysis:

    ```markdown
    # [Analysis Title]

    ## Executive summary
    [One-paragraph overview of key findings]

    ## Key findings
    - Finding 1 with supporting data
    - Finding 2 with supporting data

    ## Recommendations
    1. Specific actionable recommendation
    2. Specific actionable recommendation
    ```

- Checklists for multi-step workflows: helps the agent track progress and avoid skipping steps, especially when steps have dependencies or validation gates.
    - Example:
    ```md
    ## Form processing workflow

    Progress:
    - [ ] Step 1: Analyze the form (run `scripts/analyze_form.py`)
    - [ ] Step 2: Create field mapping (edit `fields.json`)
    - [ ] Step 3: Validate mapping (run `scripts/validate_fields.py`)
    - [ ] Step 4: Fill the form (run `scripts/fill_form.py`)
    - [ ] Step 5: Verify output (run `scripts/verify_output.py`)
    ```

- Validation loops: validate the agent's own work before moving on.
    - The pattern is: 
        - Do work
        - Run a validator (script, reference checklist, self-check)
        - Fix any issues
        - Repeat until validation passes
    - Example:
    ```md
    ## Editing workflow

    1. Make your edits
    2. Run validation: `python scripts/validate.py output/`
    3. If validation fails:
    - Review the error message
    - Fix the issues
    - Run validation again
    4. Only proceed when validation passes
    ``` 


### Effective descriptions
Principles:
- Use imperative phrasing: frame the description as an instruction to the agent: "Use this skill when..." rather than "This skill does..." The agent is deciding whether to act, so tell it when to act.
- Focus on user intent, not implementation: descript what the user is trying to achieve, not the skill's internal mechanics. The agent matches against what the user asked for.
- Err on the side of being pushy: explicitly list contexts where the skill applies, including cases where the user doesn't name the domain directly: "even if they don't explicitly mention 'CSV' or 'analysis.'"
- Keep it concise: a few sentences to a short paragraph is usually right -- long enough to cover the skill's scope, short enough that it doesn't bloat the agent's context across many skills. Limit to 1024 characters.

```yml
# Bad
description: Process CSV files.

# Good
description: >
  Analyze CSV and tabular data files — compute summary statistics,
  add derived columns, generate charts, and clean messy data. Use this
  skill when the user has a CSV, TSV, or Excel file and wants to
  explore, transform, or visualize the data, even if they don't
  explicitly mention "CSV" or "analysis."
```


### Using scripts
Skills can instruct agents to run shell commands and bundle reusable scripts in a `scripts/` directory. 

When an agent runs your script, it reads stdout and stderr to decide what to do next. A few design choices make scripts dramatically easier for agents to use:
- Avoid interactive prompts
    - Example:
    ```md
    # Bad: hangs waiting for input
    $ python scripts/deploy.py
    Target environment: _

    # Good: clear error with guidance
    $ python scripts/deploy.py
    Error: --env is required. Options: development, staging, production.
    Usage: python scripts/deploy.py --env staging --tag v1.2.3
    ```
- Document usage with `--help`: `--help` output is the primary way an agent learns your script’s interface. 
    - Include a brief description, available flags, and usage examples:
    ```
    Usage: scripts/process.py [OPTIONS] INPUT_FILE

    Process input data and produce a summary report.

    Options:
    --format FORMAT    Output format: json, csv, table (default: json)
    --output FILE      Write output to FILE instead of stdout
    --verbose          Print progress to stderr

    Examples:
    scripts/process.py data.csv
    scripts/process.py --format csv --output report.csv data.csv
    ```
- Write helpful error messages: the message should directly shape the agent's next attempt.
    - Example:
    ```
    Error: --format must be one of: json, csv, table.
       Received: "xml"
    ```
- Use structured output: prefer JSON, CSV, TSV, over free-form text
    - Example:
    ```
    # Whitespace-aligned — hard to parse programmatically
    NAME          STATUS    CREATED
    my-service    running   2025-01-15

    # Delimited — unambiguous field boundaries
    {"name": "my-service", "status": "running", "created": "2025-01-15"}
    ```
- Idempotency: Agents may retry commands. “Create if not exists” is safer than “create and fail on duplicate.”
- Input constraints: Reject ambiguous input with a clear error rather than guessing. Use enums and closed sets where possible.
- Dry-run support: For destructive or stateful operations, a `--dry-run` flag lets the agent preview what will happen.
- Meaningful exit codes: Use distinct exit codes for different failure types (not found, invalid arguments, auth failure) and document them in your `--help` output so the agent knows what each code means
- Safe defaults: Consider whether destructive operations should require explicit confirmation flags (`--confirm`, `--force`) or other safeguards appropriate to the risk level.
- Predictable output size: Many agent harnesses automatically truncate tool output beyond a threshold (e.g., 10-30K characters), potentially losing critical information. If your script might produce large output, default to a summary or a reasonable limit, and support flags like `--offset` so the agent can request more information when needed. Alternatively, if output is large and not amenable to pagination, require agents to pass an `--output` flag that specifies either an output file or `-` to explicitly opt in to stdout.


### Evaluating skills
Creating a skill and having it work doesn't validate its existence, the model could actually perform better without the skill.

Running evaluations on skills helps showcase whether the skill works reliably across varied prompts, in edge cases, and against no skill.

Creating a test case:
- Prompt: a realistic user message, something you would type.
- Expected output: a human-readable description of what success looks like.
- Input files (optional): files the skill needs to work with.

Store test cases in `evals/evals.json` inside your skill directory:
```json
{
  "skill_name": "csv-analyzer",
  "evals": [
    {
      "id": 1,
      "prompt": "I have a CSV of monthly sales data in data/sales_2025.csv. Can you find the top 3 months by revenue and make a bar chart?",
      "expected_output": "A bar chart image showing the top 3 months by revenue, with labeled axes and values.",
      "files": ["evals/files/sales_2025.csv"]
    },
    {
      "id": 2,
      "prompt": "there's a csv in my downloads called customers.csv, some rows have missing emails — can you clean it up and tell me how many were missing?",
      "expected_output": "A cleaned CSV with missing emails handled, plus a count of how many were missing.",
      "files": ["evals/files/customers.csv"]
    }
  ]
}
```
Writing good test prompts:
- Start with 2-3 test cases: dont over-invest before you've seen your first round of results; expand later.
- Vary the prompts: use different phrasings, levels of detail, and formality (e.g. "clean up this csv", and "parse the CSV at /path/to/csv, drop rows here columb B is null, and write the result to /output/path").
- Cover edge cases: include at least one prompt that tests a boundary condition: a malformed input, unusual request, or a case where the skill's instructions might be ambiguous.
- Use realistic context: real users mention file paths, column names, and personal context. Prompts like "process this data" are too vague to test anything useful.

**Core pattern -- run each test case twice:**
- Once with the skill
- Once without the skill (or a previous version)

Structure:
```
csv-analyzer/
├── SKILL.md
└── evals/
    └── evals.json
csv-analyzer-workspace/
└── iteration-1/
    ├── eval-top-months-chart/
    │   ├── with_skill/
    │   │   ├── outputs/       # Files produced by the run
    │   │   ├── timing.json    # Tokens and duration
    │   │   └── grading.json   # Assertion results
    │   └── without_skill/
    │       ├── outputs/
    │       ├── timing.json
    │       └── grading.json
    ├── eval-clean-missing-emails/
    │   ├── with_skill/
    │   │   ├── outputs/
    │   │   ├── timing.json
    │   │   └── grading.json
    │   └── without_skill/
    │       ├── outputs/
    │       ├── timing.json
    │       └── grading.json
    └── benchmark.json         # Aggregated statistics
```
The file you author by hand is `evals/evals.json`. Everything else is produced during the eval process (by agent or scripts).

**Eval runs**:
- Should start with a clean context. This ensures the agent follows only what the `SKILL.md` tells it.
- Provide:
    - Skill path (or no skill for the baseline)
    - Test prompt
    - Any input files
    - Output directory
- Example of instructions for a single with-skill run:
    ```
    Execute this task:
    - Skill path: /path/to/csv-analyzer
    - Task: I have a CSV of monthly sales data in data/sales_2025.csv.
    Can you find the top 3 months by revenue and make a bar chart?
    - Input files: evals/files/sales_2025.csv
    - Save outputs to: csv-analyzer-workspace/iteration-1/eval-top-months-chart/with_skill/outputs/
    ```
    - Use the same prompt but without the skill path when testing no skill and saving to `without_skill/outputs/`.

**Capture timing data**: when Claude finishes a task, the task completion notification includes `total_tokens` and `duration_ms`. Save those values to the output

Writing assertions: verifiable statements about what the output should contain or achieve.
- Add after seeing the first round of outputs; you often don't know what "good" looks like until after the skill has run.
- Good assertions:
    - "The output file is valid JSON" - verifiable
    - "The bar chart has labeled axes" - specific and observable
    - "The report includes at least 3 recommendations" - countable
- Bad assertions:
    - "The output is good": too vague
    - "The output uses exactly the phrase 'Total Revenue: $X'" - too briddle; correct output with different wording would fail.
- Not everything needs an assertion. Some qualities (writing style, visual design, whether the output feels right) are hard to decompose into pass/fail checks. These are better caught during human review. Reserve assertions for things that can be checked objectively.

Assertions for a test case in `evals/evals.json`:
```json
{
  "skill_name": "csv-analyzer",
  "evals": [
    {
      "id": 1,
      "prompt": "I have a CSV of monthly sales data in data/sales_2025.csv. Can you find the top 3 months by revenue and make a bar chart?",
      "expected_output": "A bar chart image showing the top 3 months by revenue, with labeled axes and values.",
      "files": ["evals/files/sales_2025.csv"],
      "assertions": [
        "The output includes a bar chart image file",
        "The chart shows exactly 3 months",
        "Both axes are labeled",
        "The chart title or caption mentions revenue"
      ]
    }
  ]
}
```

Grading outputs: evaluate each assertion against the actual outputs and record PASS or FAIL with specific evidence.
- Simple approach: give the outputs and assertions to an agent and ask it to evaluate each one.
    - Assertions that can be checked by code should use a verification script because they're deterministic
    - Grading principles:
        - **Require concrete evidence for a pass.** Don’t give the benefit of the doubt. If an assertion says “includes a summary” and the output has a section titled “Summary” with one vague sentence, that’s a FAIL — the label is there but the substance isn’t.
        - **Review the assertions themselves, not just the results.** While grading, notice when assertions are too easy (always pass regardless of skill quality), too hard (always fail even when the output is good), or unverifiable (can’t be checked from the output alone). Fix these for the next iteration.
    - Tip: for comparing skill versions, do a blind comparison by not revealing which skill came from which version. 
    - `grading.json` example:
        ```json
        {
            "assertion_results": [
                {
                "text": "The output includes a bar chart image file",
                "passed": true,
                "evidence": "Found chart.png (45KB) in outputs directory"
                },
                {
                "text": "The chart shows exactly 3 months",
                "passed": true,
                "evidence": "Chart displays bars for March, July, and November"
                },
                {
                "text": "Both axes are labeled",
                "passed": false,
                "evidence": "Y-axis is labeled 'Revenue ($)' but X-axis has no label"
                },
                {
                "text": "The chart title or caption mentions revenue",
                "passed": true,
                "evidence": "Chart title reads 'Top 3 Months by Revenue'"
                }
            ],
            "summary": {
                "passed": 3,
                "failed": 1,
                "total": 4,
                "pass_rate": 0.75
            }
        }
        ```

Aggregating results:
- Once every run in the iteration is graded, compute summary stats per config and save them to `benchmark.json` alongside the eval directories (e.g. `csv-analyzer-workspace/iter-1/benchmark.json`)
- Example:
    ```json
    {
        "run_summary": {
            "with_skill": {
                "pass_rate": { "mean": 0.83, "stddev": 0.06 },
                "time_seconds": { "mean": 45.0, "stddev": 12.0 },
                "tokens": { "mean": 3800, "stddev": 400 }
            },
            "without_skill": {
                "pass_rate": { "mean": 0.33, "stddev": 0.10 },
                "time_seconds": { "mean": 32.0, "stddev": 8.0 },
                "tokens": { "mean": 2100, "stddev": 300 }
            },
            "delta": {
                "pass_rate": 0.50,
                "time_seconds": 13.0,
                "tokens": 1700
            }
        }
    }
    ```
    - `delta` tells you what the skill costs (more time, more tokens) and what it buys (higher pass rate). A skill that adds 13 seconds but improves pass rate by 50% is probably worth it. A skill that doubles token usage for 2% improvement might not be.

Iterating on a skill:
- After grading and reviewing, you have 3 sources of signal:
    - **Failed assertions** point to specific gaps: a missing step, unclear instruction, or a case the skill doesn't handle
    - **Human feedback** points to borader quality issues: approach was wrong, output was poorly structured, skill produced a technically correct but unhelpful result.
    - **Execution transcripts** reveal why things went wrong: if the agent spent time on unproductive steps, those instructions may need to be simplified or removed.

The most effective way to turn these signals into skill improvements is to give all three — along with the current SKILL.md — to an LLM and ask it to propose changes. The LLM can synthesize patterns across failed assertions, reviewer complaints, and transcript behavior that would be tedious to connect manually. When prompting the LLM, include these guidelines:
- **Generalize from feedback.** The skill will be used across many different prompts, not just the test cases. Fixes should address underlying issues broadly rather than adding narrow patches for specific examples.
- **Keep the skill lean.** Fewer, better instructions often outperform exhaustive rules. If transcripts show wasted work (unnecessary validation, unneeded intermediate outputs), remove those instructions. If pass rates plateau despite adding more rules, the skill may be over-constrained — try removing instructions and see if results hold or improve.
- **Explain the why.** Reasoning-based instructions (“Do X because Y tends to cause Z”) work better than rigid directives (“ALWAYS do X, NEVER do Y”). Models follow instructions more reliably when they understand the purpose.
- **Bundle repeated work.** If every test run independently wrote a similar helper script (a chart builder, a data parser), that’s a signal to bundle the script into the skill’s scripts/ directory.


**Popular community skills**:
- [Modern Web Guidance](https://github.com/googlechrome/modern-web-guidance): embed web platform expertise, best practices, and browser compatibility data into coding agents.
- [Skills for Real Engineers](https://github.com/mattpocock/skills): general agent skills from Matt Pocock.
- [Ponytail](https://github.com/dietrichgebert/ponytail): reduces amount of code and token usage, provides senior-dev level reviews.


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
	- Types: command, HTTP, mcp_tool, prompt, and agent. The first three execute deterministically, latter two use Claude's judgment rather than a set of rules.
	- Low context cost because the configuration lives outside the main context window. The harness runs the handler (deterministic hook types) or makes model calls with separate windows.
    - Example: run scoped tests after every task completed in checklist (TaskCompleted), lint on every edit (PostToolUse), run all tests on tasklist completion (Stop).
        - Hook scripts that exit with code 1 get marked as completed by Claude (whether they pass or fail); code 2 means Claude should go back and fix before continuing.   
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
