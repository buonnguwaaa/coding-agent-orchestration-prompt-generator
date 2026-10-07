# Coding Agent Orchestration Prompt Generator

An agent skill that turns a coding task into a strict, copy-paste-ready **multi-agent orchestration prompt**. The generated prompt makes a parent agent act purely as a coordinator. It delegates to planner, implementer, and reviewer subagents, with explicit dependencies, safe parallelism, verbatim handoffs, scope fences, and structured output contracts.

The skill does **not** solve your task. It writes the prompt that orchestrates the agents that do.

## What you get

For each task, the skill returns:

1. A short note describing the orchestration shape.
2. One self-contained orchestration prompt (it doesn't need the skill at runtime).
3. A compact dependency diagram, when useful.

The default pipeline:

```text
Parallel planners / researchers / constraint reviewer
        │
        ▼
   Synthesis ──> Implementer ──> Independent Reviewer  (──> optional fix loop)
```

Built-in guarantees:

- **Orchestrator only:** the parent never reads code, edits files, or reviews diffs itself.
- **Dependency-first:** tasks are classified as `PARALLEL`, `SEQUENTIAL`, `GATE`, or `FINAL`. Work runs in parallel only when write sets are disjoint and no task needs another's output.
- **Verbatim handoffs:** upstream output is wrapped in delimiters such as `===== PLAN FROM PLANNER =====`. The repo and diff stay the source of truth.
- **Output contracts:**
  - Planner: scope, dependencies, `UNRESOLVED`, acceptance criteria.
  - Implementer: files changed, scope deviations, tests.
  - Reviewer: `PASS` / `FAIL` / `BLOCKED` for each criterion, with evidence.
- **Scope fences:** your exclusions are kept word for word and never softened.
- **No hallucination:** every subagent must read files before making claims about them.
- **Language matching:** output is in Vietnamese or English, matching your input. Paths, IDs, and tool params are never translated.

## Setup

The skill is called `caopg`, short for **C**oding **A**gent **O**rchestration **P**rompt **G**enerator. You don't need to clone this repo to install it.

### Claude Code (plugin marketplace)

```bash
claude plugin marketplace add buonnguwaaa/coding-agent-orchestration-prompt-generator
claude plugin install caopg@caopg
```

Inside a session, you can run `/plugin marketplace add ...` and `/plugin install caopg@caopg` instead. To get new versions, run `claude plugin marketplace update caopg`.

### Cursor, Codex, Copilot, OpenCode, and other agents

Use the [`skills`](https://github.com/vercel-labs/skills) CLI:

```bash
# Pick agents interactively
npx skills add buonnguwaaa/coding-agent-orchestration-prompt-generator

# Or target specific agents (-g installs globally rather than per project)
npx skills add buonnguwaaa/coding-agent-orchestration-prompt-generator -a cursor -a claude-code -g
```

### Manual

Copy `plugins/caopg/skills/caopg/` into your agent's skills directory, such as `.claude/skills/` or `.agents/skills/`. You can also paste the contents of `SKILL.md` into the agent's custom instructions.

## Usage

Describe the coding task and ask for an orchestration prompt. The skill triggers on requests for a prompt that coordinates planner, implementer, or reviewer subagents. You can also invoke it directly:

- `/caopg:caopg` in Claude Code when installed as a plugin
- `/caopg` when installed as a plain skill

**Example input:**

```text
Generate an orchestration prompt for this task:
- Task 1: add a `status` column to the `orders` table (migration).
- Task 2: expose `status` in GET /orders/{id}.
- Task 3: update the admin UI order detail page to show `status`.
Constraints: subagent_type=generalPurpose, run_in_background=false.
Out of scope: do not touch the payments module.
Add a fix loop, max 2 retries.
```

**Tips for better prompts:**

- **Task IDs and file paths:** list them explicitly. They are preserved exactly.
- **Tool constraints:** state any constraint such as `subagent_type=...` or `run_in_background=...`. The skill copies it verbatim. `run_in_background=false` is the default.
- **Exclusions:** write them as "do not ..." or "out of scope ...".
- **Fix loop:** ask for one if you want it, and give a max retry count. Without one, the orchestration stops on `FAIL`.
- **Project conventions:** mention any project skill files. Subagents will be told to read them.

Then paste the generated prompt into your orchestrating agent.

## Repository layout

```text
.claude-plugin/marketplace.json            # marketplace catalog (Claude Code + skills CLI)
plugins/caopg/.claude-plugin/plugin.json   # plugin manifest; bump "version" on release
plugins/caopg/skills/caopg/SKILL.md        # the skill definition
README.md
```

## Publishing a new version

1. Edit `SKILL.md`.
2. Bump `version` in `plugin.json`.
3. Validate with `claude plugin validate .`.
4. Push to GitHub.
