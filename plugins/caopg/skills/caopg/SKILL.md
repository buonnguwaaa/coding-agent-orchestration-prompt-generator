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

---

# Output language

The user may write the task in Vietnamese or in English.

Write the entire generated output in that same language:

- the short note that describes the orchestration shape;
- the copy-paste-ready orchestration prompt;
- every subagent instruction inside that prompt.

Do not mix Vietnamese and English in the generated output.

The wording in this skill is the canonical meaning. Render that meaning in the user's input language. When the input is Vietnamese, do not leave these instructions in English. When the input is English, write them in English.

Keep these unchanged in either language:

- file paths, identifiers, task IDs, code, and commands;
- tool parameter names such as `subagent_type=generalPurpose` and `run_in_background=false`;
- structural delimiter labels such as `===== PLAN FROM PLANNER =====`.

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

When implementation tasks are independent and have disjoint change surfaces, they may be parallelized explicitly. Otherwise serialize them.

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

If the user's orchestration environment has fixed tool constraints, preserve them exactly.

For example, if specified:

- `subagent_type=generalPurpose`
- `run_in_background=false`

include those exact constraints.

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
Set run_in_background=false for each Task.
Move to Step 2 only after A, B, and C have all finished.
```

Do not call a task "parallel" merely because it is convenient. The prompt should state why it is independent when that matters.

For implementation:

- parallelize only if write sets are disjoint;
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
...

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

Require this investigation rule verbatim:

```text
Never speculate about code you have not opened. If the user references a specific file, you MUST read the file before answering. Make sure to investigate and read relevant files BEFORE answering questions about the codebase. Never make any claims about code before investigating unless you are certain of the correct answer - give grounded and hallucination-free answers.
```

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

Change only the files that are necessary.
Do not make incidental changes.
Do not format or rewrite unrelated files.
Do not spawn another subagent.

Never speculate about code you have not opened. If the user references a specific file, you MUST read the file before answering. Make sure to investigate and read relevant files BEFORE answering questions about the codebase. Never make any claims about code before investigating unless you are certain of the correct answer - give grounded and hallucination-free answers.
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
You may read additional related files when needed to verify.
Do not spawn another subagent.

Never speculate about code you have not opened. If the user references a specific file, you MUST read the file before answering. Make sure to investigate and read relevant files BEFORE answering questions about the codebase. Never make any claims about code before investigating unless you are certain of the correct answer - give grounded and hallucination-free answers.
```

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
PASS / FAIL

## Fix Settings
- model: ...
- effort: ...
- reason: ...
```

`Fix Settings` is required only when the final verdict is FAIL and a fix loop exists. See section 11.

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

---

# 9. Optional fix loop

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
                         └── PASS
```

The fix agent must receive:

- original plan;
- previous implementation result;
- reviewer findings.

It must modify only the failed areas.

Set a maximum retry count when the user specifies one. If none is specified, do not invent an arbitrary retry policy; state that the orchestration stops on FAIL.

---

# 10. Speed level

The user may pick one of three levels: `fast`, `normal`, or `deep`. Vietnamese equivalents map the same way: `nhanh` → `fast`, `thường` / `bình thường` → `normal`, `sâu` / `kỹ` → `deep`.

If no level is given, use `normal`. Write the chosen level in the generated prompt and in the short note.

The level changes only how much work is spent. It never relaxes the role contracts, scope fences, handoff protocol, or the read-before-claim rule.

| Knob | `fast` | `normal` | `deep` |
|---|---|---|---|
| Planning stage | One planner. It also checks constraints. | One planner per independent area, plus a constraint reviewer when the user gave explicit constraints. Synthesis step if more than one. | Parallel researchers per area, a constraint reviewer, then a plan synthesizer. |
| Plan gate | None. | None. | A read-only plan reviewer checks the synthesized plan against the requirements before implementation. A FAIL stops the run. |
| Investigation depth | Referenced files and their direct callers. | Also call sites, related tests, and config. | Also end-to-end data flow and cross-module effects. |
| Planner output | Compact: `SCOPE`, `UNRESOLVED`, `EXPECTED FILES`, `ACCEPTANCE CRITERIA`, `EXECUTION SETTINGS`. | Full contract. | Full contract. |
| Review | One reviewer. | One reviewer. | Parallel read-only reviewers by dimension (correctness, scope, tests) plus a review synthesizer. |
| Verification | Narrowest test per changed area. Lint/typecheck only if quick. | A test per acceptance criterion, plus lint/typecheck on touched files. | Also the broader regression suite and edge cases named in `RISKS / REGRESSIONS`. |
| Model/effort bounds | Small or medium model, effort at most `medium`. | Any model, effort at most `high`. | Any model, any effort. |
| Planner/researcher defaults | Small model, `low` effort. | Medium model, `medium` effort. | Large model, `high` effort. |

Do not add a fix loop because of the level. Only the user can request one (section 9).

If the user's explicit constraints conflict with the level (for example `deep` with a fixed single reviewer), the explicit constraint wins. Mention the conflict in the short note.

---

# 11. Model and effort selection

The parent does not choose models or effort itself. Settings come from these sources, in this order of priority:

1. Constraints the user fixed explicitly. Copy them verbatim.
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
Pass model and effort exactly as given in EXECUTION SETTINGS or Fix Settings.
If a value is missing or outside the bounds for this level, use the level default instead and report that in the final answer.
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

Every subagent that inspects or describes the repository must include this rule verbatim. The parent orchestrator still does not solve the task by reading code itself; the rule applies to planner, implementer, and reviewer prompts:

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

# 16. Prompt-generation procedure

When the user gives a task, generate the final orchestration prompt using this order:

1. Parent-agent role, tool constraints, and speed level.
2. Global scope and exclusions.
3. Repository/project context.
4. Dependency graph.
5. Parallel stage(s).
6. Sequential implementation stage(s).
7. Handoff artifacts with explicit delimiters.
8. Reviewer stage.
9. Optional fix loop if requested.
10. Model and effort rules (section 11).
11. Final result contract.

Then save the prompt log file (section 15).

Preserve exact names, paths, IDs, task numbers, and constraints from the user's input.

Write the generated prompt in the same language as the user's input. See Output language.

Do not silently add business requirements.

---

# 17. Quality checklist before returning the generated prompt

Verify:

- [ ] Parent agent is clearly an orchestrator only.
- [ ] Every subagent has a clear role.
- [ ] `generalPurpose` is used when required.
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
- [ ] Reviewer has PASS/FAIL/BLOCKED criteria.
- [ ] Failures require actionable evidence.
- [ ] No subagent is asked to solve another role's responsibility.
- [ ] The generated prompt does not itself implement the underlying task.
- [ ] Planner, implementer, and reviewer prompts require reading referenced files before any claim about the code.
- [ ] The speed level is stated, and the stage shape matches it (section 10).
- [ ] Planner requires `EXECUTION SETTINGS`; reviewer requires `Fix Settings` when a fix loop exists.
- [ ] The parent passes model and effort only from user constraints, planner, reviewer, or level defaults, and never invents parameters.
- [ ] The prompt log file is saved to `prompt-logs/` (or the user's folder) in this turn, and implementers must not touch that folder.
- [ ] The generated note and prompt use the same language as the user's input, with no mixed Vietnamese and English prose.

---

# 18. Output format for this skill

Unless the user asks for explanation, return:

1. A short note describing the orchestration shape and the speed level.
2. One complete copy-paste-ready orchestration prompt.
3. If useful, a compact dependency diagram.
4. The path of the saved prompt log file (section 15).

The generated prompt itself should be self-contained and should not depend on this skill being available at runtime.

Write that output in the same language as the user's input.
