# Coding Agent Orchestration Prompt Generator

`caopg` is an agent skill that turns a coding task into a ready-to-paste prompt for a **coordinating agent**. That agent doesn't touch the code itself. It hands the work to planner, implementer, and reviewer subagents, in the right order, in parallel where it's safe, and with clear rules for what each one returns.

The skill doesn't solve your task. It writes the prompt that runs the agents who do.

## How it works

**1. The skill writes the prompt** (in your session, no code is touched):

```text
your request ──> read flags ──> estimate size ──> pick level ──> map task dependencies
                                (trivial…large)   (fast/normal/deep)          │
                                                                              ▼
                        short note + orchestration prompt + diagram + prompt-logs/<time>-<task>.md
```

**2. The coordinating agent runs it.** It only spawns subagents and passes their output along; it never reads or edits code itself.

```text
PLAN
  ├─ planner / researcher: area A ─┐
  ├─ planner / researcher: area B ─┼─ in parallel                        normal, deep
  └─ constraint reviewer ──────────┘
                  │
                  ▼
          plan synthesizer                                               when >1 planner
                  │
                  ▼
          plan reviewer ── FAIL ──> stop                                 deep only
                  │
                  │   ===== PLAN ===== + model/effort per step (passed verbatim)
                  ▼
BUILD
          implementer(s)                       parallel only if their files don't overlap
                  │
                  │   ===== PLAN ===== + ===== IMPLEMENTATION RESULT =====
                  ▼
REVIEW
          reviewer                             deep: correctness + scope + tests
                  │                                  in parallel, then a synthesizer
                  ▼
          verdict ─┬─ PASS ──────> final report
                   ├─ BLOCKED ───> stop, report what's missing
                   └─ FAIL ──────> fix round: implementer-fix ──> reviewer
                                   repeats up to --max-fix=N, then stops with the findings
                                   (no --max-fix: stop on the first FAIL)
```

Smaller tasks use less of this: `fast` runs one planner and one reviewer, and a trivial change skips the planner entirely (the implementer writes a short plan before editing).

- **Sized to the task.** A one-line rename gets an implementer and a reviewer. A cross-module migration gets parallel researchers, a plan check, and several reviewers. The coordinator can't spawn agents beyond the ones listed.
- **Safe ordering.** Work runs in parallel only when it touches different files and doesn't depend on another step's output.
- **Strict handoffs.** Each agent gets the previous agent's output word for word, and must return a fixed format. The reviewer gives `PASS` / `FAIL` / `BLOCKED` with evidence.
- **No guessing.** Every agent must read a file before making claims about it, and your exclusions are kept word for word.
- **Your language.** Write in English or Vietnamese and the prompt comes back in the same language.

## Install

**Claude Code**

```bash
claude plugin marketplace add buonnguwaaa/coding-agent-orchestration-prompt-generator
claude plugin install caopg@caopg
```

**Cursor, Codex, Copilot, OpenCode, and others** (via the `[skills](https://github.com/vercel-labs/skills)` CLI)

```bash
npx skills add buonnguwaaa/coding-agent-orchestration-prompt-generator
```

**Manual:** copy `plugins/caopg/skills/caopg/` into your agent's skills folder (such as `.claude/skills/`).

## Update

**Claude Code:** refresh the marketplace, update the plugin, then restart Claude Code.

```bash
claude plugin marketplace update caopg
claude plugin update caopg@caopg
```

`**skills` CLI:**

```bash
npx skills update caopg
```

Add `-g` for a global install or `-p` for a project install.

**Manual:** copy the latest `plugins/caopg/skills/caopg/` over your existing copy.

## Usage

Ask for an orchestration prompt and describe the task, or call the skill directly with `/caopg:caopg` (plugin) or `/caopg` (plain skill). Then paste the result into your coordinating agent.

```text
--deep --max-fix=2
Generate an orchestration prompt for this task:
- Task 1: add a `status` column to the `orders` table (migration).
- Task 2: expose `status` in GET /orders/{id}.
- Task 3: update the admin UI order detail page to show `status`.
Constraints: subagent_type=generalPurpose, run_in_background=false.
Out of scope: do not touch the payments module.
```

You get back a short note (shape, level, number of agents), the prompt, and a dependency diagram when it helps. A copy is saved to `prompt-logs/` in your project; name another folder in your request to change it.

### Flags

Flags can go anywhere in the request.


| Flag                             | What it does                                                                                  | Default               |
| -------------------------------- | --------------------------------------------------------------------------------------------- | --------------------- |
| `--fast` · `--normal` · `--deep` | The most work the agents may spend (see below). Vietnamese: `--nhanh` · `--thường` · `--sâu`. | picked from task size |
| `--max-fix=N`                    | After a `FAIL`, allow up to `N` fix-and-re-review rounds. `0` means none.                     | no fix loop           |
| `--skip-tests=true`              | Don't write or run tests. Lint, typecheck, and build still run.                               | `false`               |


### Speed levels


|                | `--fast`       | `--normal`                      | `--deep`                                          |
| -------------- | -------------- | ------------------------------- | ------------------------------------------------- |
| Planning       | 1 planner      | 1 planner per area              | parallel researchers + plan check                 |
| Review         | 1 reviewer     | 1 reviewer                      | 1 reviewer each for correctness, scope, and tests |
| Testing        | narrowest test | a test per acceptance criterion | plus the regression suite                         |
| Model / effort | up to medium   | up to high                      | no limit                                          |


Without a flag, small tasks get `fast`, medium ones `normal`, and large or risky ones (migrations, security, concurrency, payments) `deep`. The level is a ceiling: steps with nothing to do are dropped.

### Tips

- List task IDs and file paths explicitly. They're kept exactly.
- Write exclusions as "do not …" or "out of scope …".
- State tool settings like `subagent_type=...` and they're copied as is. `run_in_background=false` is the default.
- Name a model or effort for a step to override the agents' own choice.
- Mention project skill files and the agents will be told to read them.

## Development

```text
.claude-plugin/marketplace.json            # marketplace catalog
plugins/caopg/.claude-plugin/plugin.json   # plugin manifest
plugins/caopg/skills/caopg/SKILL.md        # the skill itself
```

To release: edit `SKILL.md`, bump `version` in `plugin.json`, run `claude plugin validate .`, and push.