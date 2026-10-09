---
name: caopg
description: Generate strict multi-agent orchestration prompts for coding tasks. Use when the user wants a prompt that coordinates planner, implementer, reviewer, or other subagents with explicit sequencing, parallelization, handoff artifacts, scope boundaries, and deterministic output contracts.
---
# Coding Agent Orchestration Prompt Generator

## Purpose

Generate an orchestration prompt for a parent agent whose only responsibility is to coordinate subagents.

The generated prompt must make the parent agent:

- delegate repository inspection and implementation to subagents;
- preserve explicit task dependencies;
- run independent tasks in parallel when safe;
- run dependent tasks sequentially;
- pass previous subagent outputs verbatim when they are inputs to later stages;
- prevent scope expansion;
- require structured outputs from every subagent;
- make review an independent validation stage rather than self-review by the implementer.

Do not solve the underlying coding/business task yourself. Generate the orchestration prompt.

Exception: in solo mode (section 10.2), generate a single-agent prompt instead. Sections 2, 4, 5, 8, and 9 do not apply to it.

---

# Output language

The user may write the task in Vietnamese or in English.

Write the entire generated output in that same language:

- the short note that describes the orchestration shape;
- the copy-paste-ready orchestration prompt;
- every subagent instruction inside that prompt.

Do not mix Vietnamese and English in the generated output.

The wording in this skill is the canonical meaning. Render that meaning in the user's input language. When the input is Vietnamese, do not leave these instructions in English. When the input is English, write them in English.

Keep these unchanged in either language. This is the only text that stays verbatim:

- file paths, identifiers, task IDs, code, and commands;
- tool parameter names such as `subagent_type=generalPurpose` and `run_in_background=false`;
- structural delimiter labels such as `===== PLAN FROM PLANNER =====`, and output-contract headings such as `# SCOPE` or `## Final Verdict`.

Everything else is translated, including the read-before-claim rule (section 13) and every rule block quoted in this skill. Translate faithfully: keep every condition, and keep the strength of `MUST`, `never`, and `do not`. Do not soften or shorten.

If the input mixes both languages, use the language that carries the task instructions.

---

# 1. Core orchestration model

Use this default pipeline:

```text
                         ┌── Independent planner/researcher A ──┐
                         │                                      │
Parent ── parallel ──────┼── Independent planner/researcher B ──┼──> Synthesis
                         │                                      │
                         └── Independent constraint reviewer ──┘
                                                                  │
                                                                  ▼
                                                            Implementer
                                                                  │
                                                                  ▼
                                                              Reviewer
```

This is the largest shape. Scale it down to the task size (section 10.1): drop stages that have nothing to do, and use fewer agents for simple tasks.

Only parallelize work when the tasks:

1. do not modify the same files;
2. do not require each other's outputs;
3. can be meaningfully combined afterward.

Never parallelize:

- implementation tasks with overlapping change surfaces;
- review before implementation is complete;
- tasks whose output is required by another task;
- migrations and dependent application changes when the dependency is material;
- two agents that may edit the same repository state.

Assume all implementers share one working tree. In a shared tree, run implementers one at a time, even when their write sets are disjoint: one agent's build, formatter, or diff check sees and can disturb another agent's half-finished changes.

Parallelize implementation only when both are true:

1. each implementer gets an isolated working tree (for example the user's tool offers git worktrees, such as `isolation: "worktree"` in Claude Code), or the user states that the tool supports safe concurrent edits in one tree;
2. the write sets are disjoint.

When isolated trees are used, the generated prompt must name how the results are combined (for example, one merge step that applies each branch in order and reports conflicts as BLOCKED). If the user's tool offers neither option, serialize.

Read-only stages (planners, researchers, reviewers) may run in parallel in a shared tree.

The generated parent prompt must state that `run_in_background=false` unless the user explicitly requests another mode.

---

# 2. Parent-agent role contract

Start the generated prompt with a strong role boundary:

```text
You are the coordinating agent.

You do not read source code yourself to solve the task.
You do not edit files yourself.
You do not write the implementation yourself.
You do not review the diff yourself.

Your job is to:
1. decompose the task by dependency;
2. spawn the appropriate subagent;
3. run independent tasks in parallel when it is safe;
4. wait until every dependency has finished before running a dependent task;
5. pass the previous subagent's output to the next subagent verbatim when needed;
6. synthesize the final result.

Do not take over a subagent's work.
Do not expand the scope.
```

If the user's orchestration environment has fixed tool constraints, such as `run_in_background=false`, preserve them exactly. The subagent type is chosen per role (section 2.1).

## 2.1 Subagent type per role

Each subagent runs as the agent type that fits its role. Do not give every role the same general-purpose type. A read-only role should run as a type that cannot edit files, so it cannot change the code even by mistake.

Choose each role's type from these sources, in this order:

1. **Per-role types the user names**, for example `planner=Plan, reviewer=code-reviewer`. Copy them exactly.
2. **Agent types available in the environment.** If you can see the user's list of agent types (for example Claude Code lists built-in and project agents from `.claude/agents/`), pick by role and capability. Prefer a project agent made for the role (such as a `code-reviewer` agent) over a built-in one.
3. **The defaults for the user's tool**, below.
4. **One `subagent_type` the user gave for everything**, such as `subagent_type=generalPurpose`: use it for the roles that have no type from 1–3. In the short note, say which roles fell back to it.

Defaults by role in Claude Code (`Agent` tool, parameter `subagent_type`):

| Role | Needs | `subagent_type` |
|---|---|---|
| Researcher | Read-only, find code fast | `Explore` |
| Planner, constraint reviewer, plan synthesizer, plan reviewer | Read-only, design and judge plans | `Plan` |
| Implementer, fix | Edit files, run commands | `general-purpose` |
| Reviewer, review synthesizer | Read-only, read the diff, run checks | `Plan` |

`Explore` and `Plan` cannot edit files but can run commands, so reviewers can still run `git diff` and tests.

For other tools, use the role-specific types the tool documents, if you know them. Otherwise map by capability: a read-only type for every role except implementer and fix, and an editing type for those. If you do not know any of the tool's type names and the user gave none, omit `subagent_type` and say in the short note that the user can name types per role. Never invent a type name.

The generated prompt lists each subagent run with its role and `subagent_type`, and the parent passes exactly that value. Read-only roles still keep their "do not edit" instruction, in case the type allows editing.

---

# 3. Dependency-first decomposition

Before generating concrete calls, derive a dependency graph.

Represent each task internally as:

```text
Task:
- id
- role
- reads
- writes
- depends_on
- output_required_by
- can_parallelize
```

Task IDs: when the user gives task IDs or numbers, keep them exactly. When the user gives none, assign `T1`, `T2`, … in the order the tasks appear in the request, and use those IDs unchanged in every stage. List the mapping (ID → one-line task summary) near the top of the generated prompt.

Then classify each task as:

- `PARALLEL`: independent and safe to run concurrently;
- `SEQUENTIAL`: depends on another task;
- `GATE`: must complete before implementation/review;
- `FINAL`: terminal synthesis/review.

The generated prompt should express the dependency explicitly rather than relying on vague wording such as "do these tasks in order."

Example:

```text
Dependency:
A ─┐
B ─┼──> C ──> D
E ─┘

A, B, E may run in parallel.
C starts only after A, B, E all complete.
D starts only after C completes.
```

---

# 4. Parallel tool-call rules

When parallelization is appropriate, generate a single orchestration instruction that calls all independent subagents without waiting between them.

Use wording like:

```text
Step 1. Call Task A, Task B, and Task C at the same time.
These tasks are independent. Do not wait for one another's results.
Set run_in_background=false for each Task, and give each one the subagent_type of its role.
Move to Step 2 only after A, B, and C have all finished.
```

Do not call a task "parallel" merely because it is convenient. The prompt should state why it is independent when that matters.

For implementation:

- parallelize only under the conditions in section 1 (isolated working trees, or confirmed safe concurrent edits, and disjoint write sets);
- otherwise call them sequentially.

For review:

- parallelize separate review dimensions only if they can inspect the same final state read-only;
- if multiple reviewers are parallel, add a final synthesis/reconciliation step.

NOTE: Never use placeholders or guess missing parameters in tool calls.

Example:

```text
Implementation complete
        │
        ├── Reviewer: correctness
        ├── Reviewer: scope
        └── Reviewer: tests
                 │
                 ▼
          Review synthesizer
```

---

# 5. Handoff protocol

Never embed a previous agent's output directly into a prompt without a clear boundary.

Use:

```text
===== PLAN FROM PLANNER =====
<verbatim previous output>
===== END PLAN =====
```

For multiple upstream outputs:

```text
===== RESEARCH A =====
<verbatim output A>
===== END RESEARCH A =====

===== RESEARCH B =====
<verbatim output B>
===== END RESEARCH B =====
```

Explicitly tell the downstream agent:

```text
The sections above are input from the previous subagent.
Do not treat them as the source of truth if they conflict with the current repository.
For implementation, the current repository and the task requirements are the source of truth.
```

For review:

```text
The git diff and the current repository are the source of truth.
The file list reported by the implementer is only a guide.
```

This prevents a faulty upstream report from becoming an unquestioned fact.

## 5.1 Common rules block

Rules that every subagent needs are written once in the generated prompt, in a `COMMON RULES` block. The parent prepends that block verbatim to every subagent prompt. Do not repeat these rules inside the role prompts.

```text
===== COMMON RULES =====
<read-before-claim rule, section 13>

Scope:
<scope fences, section 12>
Do not spawn another subagent.

Output:
- Use only the headings of your output format, in order.
- One line per item. No introduction, no summary, no restating of the input.
- Write `None` for an empty section.
- Quote code only as evidence, at most 5 lines per quote.

Budget: about <N> tool calls for your role (see below). When you reach it, stop and return what you have, as your role's budget rule says.
===== END COMMON RULES =====
```

Fill `<N>` per role from the level table (section 10), and give each role its budget rule:

- Planner / researcher: put whatever is still unknown under `UNRESOLVED`.
- Implementer / fix: return `Status: PARTIAL` and list unfinished work under `Remaining Issues`.
- Reviewer / synthesizer: mark every item not yet checked `[BLOCKED] budget reached`.

The budget is a soft limit to stop an agent from circling, not a target. Agents should finish well inside it.

---

# 6. Planner output contract

When generating a planning subagent prompt, require a stable structure:

```text
# SCOPE
...

# DEPENDENCIES
...

# LOCAL / PRIMARY SIDE
...

# EXTERNAL / SECONDARY SIDE
...

# SYNCHRONIZATION SIDE
...

# UNRESOLVED
...

# EXPECTED FILES
- <path>:<line> — <function / symbol> — <what changes>

# IMPLEMENTATION ORDER
...

# ACCEPTANCE CRITERIA
## Task <id>
- [ ] ...

# RISKS / REGRESSIONS
...

# EXECUTION SETTINGS
## <downstream step id> (<role>)
- model: ...
- effort: ...
- reason: ...
```

`EXECUTION SETTINGS` covers every downstream implementer and reviewer step. See section 11.

`EXPECTED FILES` gives exact locations so later agents do not explore the repository again: line numbers from the files the planner actually read, and the function or symbol at that spot. For new code, name the file and the nearest existing symbol.

`LOCAL / PRIMARY SIDE`, `EXTERNAL / SECONDARY SIDE`, and `SYNCHRONIZATION SIDE` apply only when the task spans two sides that must stay in sync (for example client and server, service and external API, or schema and application code). Otherwise the planner writes `NOT APPLICABLE — <one-line reason>` under each. Keep the headings so the format stays stable.

If some dimensions are intentionally out of scope, require the planner to state:

```text
OUT OF SCOPE
```

rather than silently ignoring them.

If something is unresolved:

```text
UNRESOLVED
- <question>
- <why it cannot be safely decided>
```

Do not let the planner silently invent requirements.

Require the read-before-claim rule from section 13.

## 6.1 Other planning-stage contracts

**Researcher** (deep, one per area). Read-only. Returns the planner contract limited to its area: `SCOPE`, `DEPENDENCIES`, `EXPECTED FILES`, `RISKS / REGRESSIONS`, `UNRESOLVED`. It does not write `ACCEPTANCE CRITERIA` or `EXECUTION SETTINGS`; the synthesizer does.

**Constraint reviewer.** Read-only. Runs only when the user gave explicit constraints (section 10). Returns:

```text
# CONSTRAINTS
## <constraint, quoted from the user>
- Affected areas / files:
- What the plan must do to respect it:
- Conflicts with other requirements: None / ...
```

**Plan synthesizer.** Runs when there is more than one planner or researcher. Receives every upstream output verbatim, each in its own delimiter block. Returns the full planner contract (section 6) for the whole task, plus:

```text
# CONFLICTS
- <what the inputs disagreed on> — Resolved: <how, and from which input> / Unresolved: moved to UNRESOLVED
```

It must not drop an upstream `UNRESOLVED` item or a `CONSTRAINTS` entry. It may only merge duplicates.

**Plan reviewer** (deep plan gate). Read-only. Receives the user's request and the synthesized plan. Returns:

```text
# PLAN REVIEW
## Requirements
- [PASS] / [FAIL] <requirement or constraint> — <evidence>
## Execution settings
- Unchanged / Adjusted: <step> → <model, effort, reason>
## Final Verdict
PASS / FAIL
```

FAIL means a requirement or constraint from the user is missing, contradicted, or out of scope in the plan. The parent stops on FAIL and reports the findings. It does not re-plan.

---

# 7. Implementer output contract

The implementation subagent must receive the planner output verbatim.

Start its prompt with:

```text
===== PLAN FROM PLANNER =====
...
===== END PLAN =====
```

Then require:

```text
Implement exactly the plan above.

The plan is the scope contract.
Do not expand the scope yourself.
Do not resolve UNRESOLVED items by guessing.
If you find a blocker or a contradiction:
- do not expand the scope to work around the blocker;
- record the blocker;
- continue the unaffected parts if that is safe.

Start from the locations in EXPECTED FILES. Read other files only when the change needs them (callers, types, config).

Change only the files that are necessary.
Do not make incidental changes.
Do not format or rewrite unrelated files.
```

Before finishing, require the implementer to inspect its own final change surface:

```text
Before you finish:
1. inspect the git diff;
2. remove out-of-scope changes;
3. confirm the files that were actually changed;
4. confirm that tests or verification were run, or state clearly that they were not run.
```

Output:

```text
# IMPLEMENTATION RESULT

## Status
COMPLETE / PARTIAL / BLOCKED

## Files Changed
- ...

## Task <id>
- Implemented:
- Verification:

## Remaining Issues
- None / ...

## Scope Deviations
- None / ...

## Tests
- ...
```

`PARTIAL` means some tasks are done and others hit a blocker listed under `Remaining Issues`. `BLOCKED` means nothing safe could be changed. The parent runs the reviewer after `COMPLETE` or `PARTIAL`. After `BLOCKED`, it skips review and goes straight to the final report (section 16).

---

# 8. Reviewer contract

The reviewer must be independent from the implementer.

Give it:

1. the original plan;
2. the implementer's report;
3. the current repository/diff.

Use explicit boundaries:

```text
===== PLAN =====
...
===== END PLAN =====

===== IMPLEMENTER RESULT =====
...
===== END IMPLEMENTER RESULT =====
```

Then:

```text
Evaluate the current repository state.
Do not edit code.
Do not accept the implementer's report as evidence.
The git diff and the current repository are the source of truth.
The file list reported by the implementer is only a guide.
Start from the git diff. Read the changed files and their direct callers. Open other files only when a specific finding needs it.
```

At `deep`, a dimension reviewer may go beyond direct callers when its dimension needs it.

Require per-criterion verdicts:

```text
# REVIEW

## Task <id>
- [PASS] ...
- [FAIL] ...
- [BLOCKED] ...

## Scope
- [PASS] / [FAIL]

## Regression
- [PASS] / [FAIL]

## Tests
- [PASS] / [FAIL] / [BLOCKED]

## Final Verdict
PASS / FAIL / BLOCKED

## Fix Settings
- model: ...
- effort: ...
- reason: ...
```

`Fix Settings` is required only when the final verdict is FAIL and a fix loop exists. See section 11.

The final verdict follows these rules, in order:

1. `FAIL` if any item is `[FAIL]`.
2. Otherwise `BLOCKED` if any item is `[BLOCKED]`: it cannot be judged because something outside the task is missing (an unresolved requirement, a missing dependency or credential, an environment that cannot run the check).
3. Otherwise `PASS`.

`[BLOCKED]` items stay listed even when the verdict is FAIL.

What the parent does with the verdict:

- `PASS`: final report.
- `FAIL`: start a fix round if one is left (section 9); otherwise final report.
- `BLOCKED`: final report. Never start a fix round for BLOCKED; a fix cannot supply what is missing.

For every failure require:

```text
- File:
- Location:
- Problem:
- Expected:
- Actual:
- Why it violates the plan:
```

Do not let the reviewer fix the issue unless the user explicitly asks for a fix loop.

## 8.1 Review synthesizer

Runs when several reviewers check different dimensions in parallel (deep). Read-only. It receives every reviewer output verbatim, each in its own delimiter block, plus the plan and the implementer result.

It returns the same `# REVIEW` format as a single reviewer, with these rules:

- Keep every `[FAIL]` and `[BLOCKED]` finding from every reviewer, with its evidence. Merge duplicates.
- It may drop a finding only after checking the repository and showing the finding is wrong. List each dropped finding under `## Rejected Findings` with the reason.
- Apply the final-verdict rules from section 8 to the merged findings. One accepted `[FAIL]` from any reviewer makes the verdict `FAIL`.
- Write `Fix Settings` when the verdict is FAIL and a fix loop exists.

---

# 9. Optional fix loop

The user can request a fix loop in words (for example "add a fix loop, max 2 retries") or with a flag, anywhere in the request:

- `--max-fix=N` or `--max-fix N`, where `N` is a whole number. Aliases: `--fix=N`, `--sửa=N`, `--sua=N`.

Match the flag case-insensitively and remove it from the task text before using it.

- `N` ≥ 1: add a fix loop with at most `N` fix rounds. One round is one Implementer-Fix run followed by one Reviewer run.
- `N` = 0: no fix loop. The orchestration stops on FAIL, even if the task text asks for one.
- If `N` is missing or not a whole number, do not guess. Generate no fix loop and say in the short note that the flag was ignored.
- If both the flag and the task text give a count, the flag wins. Mention the conflict in the short note. If the flag is given more than once, use the last one.

If the user wants an iterative repair workflow, generate:

```text
Planner
   ↓
Implementer
   ↓
Reviewer
   ↓
FAIL ──> Implementer-Fix ──> Reviewer
                         │
                         ├── PASS ──> final report
                         ├── FAIL ──> next round, or final report when no rounds are left
                         └── BLOCKED ──> final report
```

The fix agent must receive:

- original plan;
- previous implementation result;
- reviewer findings.

It must modify only the failed areas.

Set a maximum retry count when the user specifies one. If none is specified, do not invent an arbitrary retry policy; state that the orchestration stops on FAIL.

The generated parent prompt must count fix rounds. When the reviewer still returns FAIL after the last allowed round, the parent stops, does not start another fix, and reports the final FAIL with the last reviewer findings and the number of rounds used.

---

# 10. Speed level

The user may pick one of three levels: `fast`, `normal`, or `deep`. The level is given as a flag prefixed with `--`, anywhere in the request:

- `--fast`, `--nhanh` → `fast`
- `--normal`, `--thường`, `--bình-thường`, `--thuong`, `--binh-thuong` → `normal`
- `--deep`, `--sâu`, `--kỹ`, `--sau`, `--ky` → `deep`

Match flags case-insensitively. A bare word without `--` (for example `deep` or `nhanh` in the task text) is part of the task, not a level. Remove the flag from the task text before using it.

If more than one level flag is given, use the last one and mention it in the short note. Other flags (such as `--max-fix`, section 9, `--solo`, section 10.2, and `--skip-tests`, section 14.1) are not levels. An unknown `--` flag is not a level; leave it in the task text.

If no level flag is given, pick the level from the task size (section 10.1). Write the chosen level in the generated prompt and in the short note.

The level changes only how much work is spent. It never relaxes the role contracts, scope fences, handoff protocol, or the read-before-claim rule.

| Knob                        | `fast`                                                                                                | `normal`                                                                                                                             | `deep`                                                                                                                    |
| --------------------------- | ------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Planning stage              | One planner. It also checks constraints.                                                                | One planner per independent area, plus a constraint reviewer when the user gave explicit constraints. Synthesis step if more than one. | Parallel researchers per area, a constraint reviewer, then a plan synthesizer.                                              |
| Plan gate                   | None.                                                                                                   | None.                                                                                                                                  | A read-only plan reviewer checks the synthesized plan against the requirements before implementation. A FAIL stops the run. |
| Investigation depth         | Referenced files and their direct callers.                                                              | Also call sites, related tests, and config.                                                                                            | Also end-to-end data flow and cross-module effects.                                                                         |
| Planner output              | Compact:`SCOPE`, `UNRESOLVED`, `EXPECTED FILES`, `ACCEPTANCE CRITERIA`, `EXECUTION SETTINGS`. | Full contract.                                                                                                                         | Full contract.                                                                                                              |
| Review                      | One reviewer.                                                                                           | One reviewer.                                                                                                                          | Parallel read-only reviewers by dimension (correctness, scope, tests) plus a review synthesizer.                            |
| Verification                | Narrowest test per changed area. Lint/typecheck only if quick. The reviewer does not rerun them (section 14).                                          | A test per acceptance criterion, plus lint/typecheck on touched files.                                                                 | Also the broader regression suite and edge cases named in`RISKS / REGRESSIONS`.                                           |
| Model/effort bounds         | Small or medium model, effort at most`medium`.                                                        | Any model, effort at most`high`.                                                                                                     | Any model, any effort.                                                                                                      |
| Planner/researcher defaults | Small model,`low` effort.                                                                             | Medium model,`medium` effort.                                                                                                        | Large model,`high` effort.                                                                                                |
| Tool-call budget per subagent | Planner 15, implementer 30, reviewer 15. | Planner 30, implementer 60, reviewer 30. | Researcher 40, implementer 100, each reviewer 40. |
| Generated prompt length | Aim for at most about 60 lines. | Aim for at most about 120 lines. | No target, but no padding. |

Do not add a fix loop because of the level. Only the user can request one (section 9).

**Explicit constraints** are restrictions the user puts on how the work may be done, for example "do not change the public API", "use library X", "no new dependencies", or "keep backward compatibility". They decide whether a constraint reviewer runs. These are not explicit constraints:

- requirements and acceptance criteria (what the result must do);
- scope exclusions such as "do not touch the payments module" (section 12 handles them);
- tool and orchestration settings such as `subagent_type` (section 2.1), `run_in_background`, model, or effort.

If an explicit user instruction about the orchestration conflicts with the level (for example `deep` with a fixed single reviewer), the user's instruction wins. Mention the conflict in the short note.

## 10.1 Sizing the workforce

Spend no more agents than the task needs. Every subagent run costs time, and a long chain of agents for a simple change makes the session run far longer than the work itself.

Before choosing the stage shape, estimate the task size from the request:

| Size      | Signals                                                                                                                                 | Subagent runs (without fix loop) |
| --------- | ----------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| `trivial` | One area, about 1–2 files, mechanical change with a clear target (rename, text or config value, add a field that is only passed through). | 1 (solo) or 2                    |
| `small`   | One area, a few files, clear requirements, no risk markers.                                                                                | 3                                |
| `medium`  | 2–3 independent areas, or requirements that need investigation to pin down.                                                               | up to 6                          |
| `large`   | Many areas, or any risk marker: data migration, concurrency, security, auth, payments, cross-module contracts, or possible data loss.    | as the `deep` shape needs        |

When the task sits between two sizes, choose the smaller one, unless it has a risk marker.

Without a level flag, map size to level: `trivial` / `small` → `fast`, `medium` → `normal`, `large` → `deep`. A `trivial` task without a level flag runs in solo mode (section 10.2), unless `--solo=false` or a fix loop was requested.

With a level flag, the level is the user's choice and is kept. It sets the most work allowed, not the least. Inside it, still drop every stage that has nothing to do:

- One independent area: one planner (or one researcher), and no synthesis step.
- No explicit user constraints: no constraint reviewer.
- Only one implementation task: one implementer.

Shape per size:

- `trivial` in solo mode: one agent, no coordinator (section 10.2).
- `trivial` at `fast` (explicit `--fast`, `--solo=false`, or a fix loop): no planner. One implementer first writes a compact plan in its output (`SCOPE`, `EXPECTED FILES`, `ACCEPTANCE CRITERIA`), then makes the change. One independent reviewer checks it against that plan and the user's request. Implementer settings come from the level defaults. If the implementer finds the task is bigger than expected (more files or areas than the request implies, or a risk marker), it stops with BLOCKED and says why, instead of growing the change.
- `small`: planner → implementer → reviewer.
- `medium` and `large`: the shape from the level table, minus the empty stages above.

The `trivial`-at-`fast` implementer writes `file:line` locations in its compact plan, like `EXPECTED FILES` (section 6), so the reviewer can start from them.

The generated parent prompt must list the exact subagent runs it will make and tell the parent not to spawn any others. Fix rounds (section 9) are the only allowed extra runs.

State the size, the reason in one line, and the number of subagent runs in the short note. Example: `Size: small (one endpoint, 2 files) — 3 runs: planner, implementer, reviewer; up to 2 fix rounds.`

## 10.2 Solo mode

In solo mode, one agent makes the change and checks its own work. There is no coordinator and no subagent. It is the fastest shape, but it gives up the independent review.

The user controls it with a flag, anywhere in the request:

- `--solo=true`: always solo.
- `--solo=false`: never solo.
- Not given: solo only for a `trivial` task with no level flag and no fix loop requested (section 10.1).

Match the flag name and value case-insensitively and remove the flag from the task text. If the value is anything other than `true` or `false`, treat the flag as not given and say so in the short note. If the flag is given more than once, use the last one.

With `--solo=true` on a task that is not `trivial` or `small`, keep solo mode, but warn in the short note that there is no independent review. For a task with a risk marker, name the risk marker in the warning.

In solo mode:

- `--max-fix` does not apply. The agent fixes its own mistakes inside its budget. Mention in the short note that the flag was ignored.
- The level (given or picked) still sets investigation depth, verification, and the budget. Use the implementer budget plus the reviewer budget from the level table.
- `--skip-tests` works as usual (section 14.1).
- There are no model or effort settings. The user's own agent runs the prompt.

The solo prompt contains, in this order:

1. Role: "You are the only agent for this task. Do not spawn subagents."
2. Task IDs, scope fences, and explicit constraints, kept as the user wrote them.
3. The read-before-claim rule (section 13) and the output rules from section 5.1.
4. Steps:
   1. Read the referenced files. Write a short plan: `SCOPE`, `EXPECTED FILES` with `file:line` locations, `ACCEPTANCE CRITERIA`. If the task turns out bigger than expected or has a risk marker, stop and return `BLOCKED` with the reason.
   2. Make the change. Change only what the plan needs.
   3. Verify as the level and `--skip-tests` say (section 14).
   4. Check your own `git diff` against each acceptance criterion and the scope fences. Remove unrelated changes. Fix what fails, within the budget.
5. Output:

```text
# RESULT

## Verdict
PASS / FAIL / BLOCKED (self-checked)

## Plan
- Scope:
- Locations:

## Acceptance Criteria
- [PASS] / [FAIL] / [BLOCKED] <criterion> — <evidence>

## Files Changed
- ...

## Tests and Verification
- ...

## Unresolved and Blocked
- None / ...
```

State `Mode: solo` and the reason in the short note.

## 10.3 Prompt length

Every line of the generated prompt is read by the coordinator and by every subagent that gets it. Keep it short:

- Stay near the length target in the level table. The user's own task text and quoted constraints do not count toward it.
- At `fast`, include only the compact output formats. Leave out the side headings (`LOCAL / PRIMARY SIDE` and the others), sections that do not apply, and the `CONFLICTS` and `Rejected Findings` sections.
- Write each shared rule once, in `COMMON RULES` (section 5.1).
- No examples, no explanations of why a rule exists, and no restating of the dependency graph in prose. For a single chain, one line such as `Planner → Implementer → Reviewer` is enough.

---

# 11. Model and effort selection

The parent does not choose models or effort itself. Settings come from these sources, in this order of priority:

1. Settings the user fixed explicitly. Copy them verbatim. They are authoritative, even outside the level's bounds.
2. The planner's `EXECUTION SETTINGS` for implementer and reviewer steps. When there are multiple planners, the synthesis step merges them; when there is a plan gate, the plan reviewer may adjust them.
3. The reviewer's `Fix Settings` for the next fix step.
4. The level defaults from section 10 for planners, researchers, and any step that has no setting above.

Require the planner to choose per downstream step, based on how hard and risky that step is:

```text
For every downstream implementer and reviewer step, choose a model and an effort level and give a one-line reason.
Use a smaller model and lower effort for mechanical, local changes.
Use a larger model and higher effort for cross-module logic, data migrations, concurrency, security, or unclear requirements.
Stay within these bounds: <level bounds>.
```

Require the reviewer, on FAIL with a fix loop, to choose `Fix Settings` the same way. Escalate model or effort when the failure came from complexity rather than a simple slip.

The generated parent prompt must say:

```text
Settings the user fixed explicitly always win. Pass them exactly, even when they are outside this level's bounds, and note that in the final report.
For other steps, pass model and effort exactly as given in EXECUTION SETTINGS or Fix Settings.
If a value from EXECUTION SETTINGS or Fix Settings is missing or outside the bounds for this level, use the level default instead and note that in the final report.
If the subagent tool has no model or effort parameter, omit it. Do not invent parameters.
```

Name the parameters the way the user's tool does, when the user states it. In Claude Code the `Agent` tool takes `model` (`haiku`, `sonnet`, `opus`) and `effort` (`low`, `medium`, `high`, `xhigh`, `max`). Otherwise use the neutral tiers small / medium / large and low / medium / high.

---

# 12. Scope fences

Always preserve user-provided exclusions verbatim.

Group them into:

```text
GLOBAL SCOPE
...

EXPLICITLY OUT OF SCOPE
...

DO NOT TOUCH
...
```

Do not weaken exclusions while rewriting.

If a user says "do not do X", never transform it into a softer instruction such as "prefer not to do X."

---

# 13. File and repository safety

For implementation prompts, prefer:

```text
Do not create a new file if an existing file can satisfy the requirement,
unless the plan explicitly requires a new file.

Do not change a public API, config, or schema outside the scope.

Do not refactor unrelated code merely because you noticed an improvement opportunity.
```

Only include these when compatible with the user's task.

Do not invent repository conventions. If the user supplies project-specific skill files, tell the subagent to read them.

Every subagent that inspects or describes the repository must include this read-before-claim rule: planners, researchers, constraint reviewers, plan reviewers, implementers, reviewers, and synthesizers. The parent orchestrator still does not solve the task by reading code itself.

It goes in the `COMMON RULES` block (section 5.1), so it is written once per generated prompt. In solo mode it goes in the single prompt. This is the canonical English text. In English output, copy it exactly. In Vietnamese output, translate it faithfully (see Output language):

```text
Never speculate about code you have not opened. If the user references a specific file, you MUST read the file before answering. Make sure to investigate and read relevant files BEFORE answering questions about the codebase. Never make any claims about code before investigating unless you are certain of the correct answer - give grounded and hallucination-free answers.
```

---

# 14. Tests and verification

Do not merely say "write tests."

Generate explicit verification expectations:

```text
For each acceptance criterion:
- identify the relevant test;
- run the narrowest useful test first;
- report the command and result;
- distinguish "passed", "not run", and "blocked".
```

Do not claim tests passed unless the subagent actually ran them.

Run each test once. At `fast`, the reviewer does not rerun tests, lint, typecheck, or build. It checks the commands and results the implementer reported against the diff. It reruns a check only if the result is missing, failed, or does not match the diff (for example, the test does not cover the changed code). At `normal` and `deep`, the reviewer reruns what it needs to judge each criterion.

## 14.1 Skipping tests

The user can turn tests off with a flag, anywhere in the request:

- `--skip-tests=true` skips tests.
- `--skip-tests=false` runs tests as usual.
- If the flag is not given, the value is `false`.

Match the flag name and value case-insensitively and remove the flag from the task text before using it. If the value is anything other than `true` or `false`, use `false` and say in the short note that the flag was ignored. If the flag is given more than once, use the last one.

With `--skip-tests=true`, the generated prompt must say:

- No subagent runs tests.
- No subagent writes or edits tests unless the task explicitly asks for that test work (see below). All other existing tests are left untouched.
- The implementer still runs lint, typecheck, or build on touched files when the project has them and they are quick. They catch broken code at little cost. Report them under `Verification`.
- The implementer and reviewer write `Not run — skipped by the user (--skip-tests=true)` under `Tests`.
- The reviewer still reads the diff and checks correctness, scope, and regressions by reading the code. It must not return FAIL because tests are missing or were not run. It may name untested risks as notes.
- At `deep`, there is no separate tests reviewer.
- Acceptance criteria are checked by reading the code and by lint, typecheck, or build output, not by tests.

The flag turns off test execution and unrequested test work. It does not remove work the task asks for. If the task explicitly asks for tests (for example "add a unit test for X"), writing those tests stays in scope: the implementer writes them, the reviewer checks them by reading them, and nobody runs them. Mention this in the short note.

State `Tests: skipped (--skip-tests=true)` in the short note.

---

# 15. Prompt log file

As soon as the prompt is generated, in the same turn, save it to a file in the current project. Do this yourself, before returning the answer. It is not an instruction inside the generated prompt, and it does not wait for the prompt to be executed.

Location:

- Folder: `prompt-logs/` at the current project root (your working directory), unless the user names another folder. Create it if missing.
- File name: `<YYYYMMDD-HHMMSS>-<task-slug>.md`. Get the local time from the environment (for example `date +%Y%m%d-%H%M%S`). The slug is a short ASCII kebab-case summary of the task. If the time cannot be read, use `<task-slug>-<n>.md` with the next free `n`.
- Never overwrite an existing file.

File content, in this order:

```text
# <task title>

- Created: <YYYY-MM-DD HH:MM:SS>
- Level: <fast | normal | deep>
- Size: <trivial | small | medium | large> — <number> subagent runs
- Tests: <run | skipped (--skip-tests=true)>
- Max fix rounds: <N | none>
- Mode: <orchestrated | solo>

## Request
<the user's request, verbatim>

## Note
<the short note>

## Dependency diagram
<the diagram, if any>

## Orchestration prompt
<the complete generated prompt, verbatim, in one fenced text block>
```

Rules:

- The file holds exactly what you returned in the answer. Do not paraphrase or shorten it.
- Write only this file. Do not edit `.gitignore` or anything else. Mention once in the note that the user may want to ignore `prompt-logs/`.
- End the answer with the saved file path.
- If you cannot write files in this environment, say so in one line and still return the prompt.

Every implementer and fix prompt in the generated orchestration must list `prompt-logs/` (or the user's folder) under DO NOT TOUCH.

---

# 16. Final result contract

The generated parent prompt must end with the exact format of the parent's final report. The parent builds it only from subagent outputs. It adds no claims about the code of its own.

```text
# FINAL RESULT

## Verdict
PASS / FAIL / BLOCKED / STOPPED AT PLAN GATE

## Stages
- <run id> <role>: <done | skipped | stopped> — <status or verdict it returned>

## Files Changed
- <from the last implementer result; mark any file the reviewer found differently>

## Tests and Verification
- Tests: <ran: commands and results | not run: reason | skipped by the user (--skip-tests=true)>
- Lint / typecheck / build: <commands and results | not run: reason>

## Unresolved and Blocked
- None / <item> — <from which stage>

## Fix Rounds
- <used> of <max> / no fix loop

## Settings Notes
- None / <step>: <user setting outside level bounds | fallback to level default, and why>
```

Write an empty section as `None` on one line. At `fast`, use the short form: `Verdict`, `Files Changed`, `Tests and Verification`, and `Unresolved and Blocked`. Add `Fix Rounds` and `Settings Notes` only when they are not empty. Leave out `Stages`.

---

# 17. Prompt-generation procedure

First decide the mode (section 10.2). In solo mode, write the solo prompt from section 10.2 instead of the steps below, then save the prompt log.

When the user gives a task, generate the final orchestration prompt using this order:

1. Parent-agent role, tool constraints, speed level, task size, and the list of subagent runs with each run's role and `subagent_type` (sections 2.1, 10.1).
2. The `COMMON RULES` block (section 5.1), and the instruction to prepend it to every subagent prompt. Scope fences go inside it.
3. Global scope in one or two lines for the parent. The full scope fences are in `COMMON RULES`; do not repeat them.
4. Repository/project context.
5. Dependency graph.
6. Parallel stage(s).
7. Sequential implementation stage(s).
8. Handoff artifacts with explicit delimiters.
9. Reviewer stage.
10. Optional fix loop if requested.
11. Model and effort rules (section 11).
12. Final result contract (section 16).

Then save the prompt log file (section 15).

Preserve exact names, paths, IDs, task numbers, and constraints from the user's input.

Write the generated prompt in the same language as the user's input. See Output language.

Do not silently add business requirements.

---

# 18. Quality checklist before returning the generated prompt

Verify:

In solo mode, check only that the prompt has the parts listed in section 10.2, keeps the scope fences and the read-before-claim rule, matches the user's language, and that the short note states `Mode: solo` (with a warning when needed). Skip the items about the parent, handoffs, and separate reviewers.

For orchestrated mode:

- [ ] Parent agent is clearly an orchestrator only.
- [ ] Every subagent has a clear role.
- [ ] Each subagent run has the `subagent_type` of its role (section 2.1): read-only types for planning and review roles, an editing type for implementer and fix. A single user-given type is only a fallback, and the note names the roles that use it.
- [ ] `run_in_background=false` is used when required.
- [ ] Independent tasks are parallelized where safe.
- [ ] Dependent tasks are explicitly sequential.
- [ ] No reviewer runs before implementation completes.
- [ ] Previous outputs are passed verbatim where required.
- [ ] Handoff sections have clear delimiters.
- [ ] Repository/diff is treated as source of truth during review.
- [ ] Scope exclusions are preserved.
- [ ] Planner has a structured output contract.
- [ ] Implementer has a structured output contract.
- [ ] Reviewer has PASS/FAIL/BLOCKED criteria and a PASS/FAIL/BLOCKED final verdict, and the parent's action for each verdict is stated (section 8).
- [ ] Every synthesizer, constraint reviewer, and plan reviewer used has its output contract, and the review synthesizer keeps every accepted FAIL (sections 6.1, 8.1).
- [ ] Task IDs are the user's, or generated `T1`, `T2`, … listed once and used unchanged (section 3).
- [ ] Implementers run in parallel only with isolated working trees (or confirmed safe concurrent edits) and disjoint write sets (section 1).
- [ ] A constraint reviewer runs only for explicit constraints as defined in section 10, not for ordinary requirements.
- [ ] The parent's final report uses the format from section 16 (short form at `fast`).
- [ ] Shared rules appear once, in `COMMON RULES`, with a tool-call budget and budget rule per role (section 5.1).
- [ ] The planner gives `file:line` locations, the implementer starts from them, and the reviewer starts from the diff (sections 6–8).
- [ ] At `fast`, the reviewer does not rerun checks the implementer already ran, unless the result is missing, failed, or does not match the diff (section 14).
- [ ] The prompt is near the level's length target and has no sections that do not apply (section 10.3).
- [ ] Failures require actionable evidence.
- [ ] No subagent is asked to solve another role's responsibility.
- [ ] The generated prompt does not itself implement the underlying task.
- [ ] Every subagent that reads the repository has the read-before-claim rule, translated faithfully when the output is Vietnamese (section 13).
- [ ] The speed level is stated, and the stage shape matches it (section 10).
- [ ] With `--skip-tests=true`, no subagent runs tests or writes unrequested tests, the reviewer does not FAIL for missing tests, and the note says tests are skipped (section 14.1).
- [ ] The task size is stated, no stage exists that has nothing to do, and the parent is told not to spawn subagents beyond the listed runs (section 10.1).
- [ ] Planner requires `EXECUTION SETTINGS`; reviewer requires `Fix Settings` when a fix loop exists.
- [ ] If a fix loop exists, the max fix rounds come from `--max-fix` or the task text, and the parent stops with a final FAIL report once they run out (section 9).
- [ ] The parent passes model and effort only from user settings, planner, reviewer, or level defaults, never invents parameters, and lets explicit user settings win over level bounds (section 11).
- [ ] The prompt log file is saved to `prompt-logs/` (or the user's folder) in this turn, and implementers must not touch that folder.
- [ ] The generated note and prompt use the same language as the user's input, with no mixed Vietnamese and English prose.

---

# 19. Output format for this skill

Unless the user asks for explanation, return:

1. A short note describing the mode, the orchestration shape, the speed level, and the task size with its number of subagent runs.
2. One complete copy-paste-ready orchestration prompt, or the solo prompt in solo mode.
3. If useful, a compact dependency diagram.
4. The path of the saved prompt log file (section 15).

The generated prompt itself should be self-contained and should not depend on this skill being available at runtime.

Write that output in the same language as the user's input.
