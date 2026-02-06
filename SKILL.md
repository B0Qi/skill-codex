---
name: codex
description: Use when the user asks to run Codex CLI (codex exec, codex resume) or references OpenAI Codex for code analysis, refactoring, or automated editing
---

# Codex Skill Guide

## Collaboration Paradigm (Three-Phase Workflow)
This skill is designed to work with **Claude Code ↔ Codex** collaboration. Enforce the phase boundaries.

1. **Explore (Claude Code)**
   - Read-only research and context gathering.
   - Clarify goals, constraints, and acceptance criteria.
   - Produce a standardized Task Package for Codex.
   - No file edits.

2. **Execute (Codex)**
   - Codex acts only on the provided Task Package.
   - Writes are allowed only in files explicitly listed in the package.
   - If required context is missing, stop and ask for clarification.

3. **Accept (Claude Code)**
   - Perform diff review and run tests if specified.
   - Verify acceptance criteria.
   - Decide accept, revise, or request further iteration.

## Task Package Format (Standard Prompt Template)
Codex must receive a normalized Task Package. If missing, ask Claude Code to supply it.

```
[Task Package]
Goal:
- <single-sentence goal>

Files:
- <explicit file list>

Constraints:
- <non-functional requirements, style rules, time limits, deps>
- <do NOT change list>

Acceptance Criteria:
- <observable outcomes>
- <tests to run or checks to pass>

Notes:
- <assumptions, risk areas, or context snapshots>
```

## Running a Task
1. Confirm the Task Package exists and is complete.
2. Ask the user (via `AskUserQuestion`) which model to run (`gpt-5.3-codex` or `gpt-5.3`) AND which reasoning effort to use (`xhigh`, `high`, `medium`, or `low`) in a **single prompt with two questions**.
3. Select the sandbox mode required for the task; default to `--sandbox read-only` unless edits or network access are necessary.
4. Assemble the command with the appropriate options:
   - `-m, --model <MODEL>`
   - `--config model_reasoning_effort="<xhigh|high|medium|low>"`
   - `--sandbox <read-only|workspace-write|danger-full-access>`
   - `--full-auto`
   - `-C, --cd <DIR>`
   - `--skip-git-repo-check`
5. Always use `--skip-git-repo-check`.
6. When continuing a previous session, use `codex exec --skip-git-repo-check resume --last` via stdin. When resuming, don't use any configuration flags unless explicitly requested by the user (e.g., model or reasoning effort). Resume syntax:
   - `echo "your prompt here" | codex exec --skip-git-repo-check resume --last 2>/dev/null`
   - All flags have to be inserted between `exec` and `resume`.
7. **IMPORTANT**: By default, append `2>/dev/null` to all `codex exec` commands to suppress thinking tokens (stderr). Only show stderr if the user explicitly requests to see thinking tokens or if debugging is needed.
8. Run the command, capture stdout/stderr (filtered as appropriate), and summarize the outcome for the user.
9. **After Codex completes**, inform the user: "You can resume this Codex session at any time by saying 'codex resume' or asking me to continue with additional analysis or changes."

## Smart Sandbox Selection
Choose the minimum required access. Prefer least privilege.

| Use case | Sandbox mode | Key flags |
| --- | --- | --- |
| Read-only review or analysis | `read-only` | `--sandbox read-only 2>/dev/null` |
| Apply local edits | `workspace-write` | `--sandbox workspace-write --full-auto 2>/dev/null` |
| Permit network or broad access | `danger-full-access` | `--sandbox danger-full-access --full-auto 2>/dev/null` |
| Resume recent session | Inherited from original | `echo "prompt" \| codex exec --skip-git-repo-check resume --last 2>/dev/null` |
| Run from another directory | Match task needs | `-C <DIR>` plus other flags `2>/dev/null` |

If unsure, default to `read-only` and ask if writes/network are needed.

## Acceptance Workflow (Claude Code)
After Codex execution, Claude Code should:

1. Perform diff review of modified files.
2. Run tests or checks listed in `Acceptance Criteria`.
3. Verify constraints and confirm no forbidden files were modified.
4. Decide one of:
   - **Accept** — proceed to next task or commit
   - **Revise** — re-enter Execute phase with feedback
   - **Abort** — revert changes

## Conflict Avoidance (Role Locking)
To avoid edit conflicts and context drift:

- **Explore phase**: Claude Code is read-only. Codex must not write.
- **Execute phase**: Codex writes; Claude Code does not change files.
- **Accept phase**: Claude Code reviews and validates; Codex does not write unless explicitly re-entering Execute phase.

If a phase boundary is unclear, stop and request clarification.

## Following Up
- If the task package is incomplete, request clarification **before** running Codex.
- After every `codex` command, use `AskUserQuestion` to confirm next steps, collect clarifications, or decide whether to resume with `codex exec resume --last`.
- Restate the chosen model, reasoning effort, and sandbox mode when proposing follow-up actions.

## Error Handling
- Stop and report failures whenever `codex --version` or a `codex exec` command exits non-zero; request direction before retrying.
- Before you use high-impact flags (`--full-auto`, `--sandbox danger-full-access`, `--skip-git-repo-check`) ask the user for permission using AskUserQuestion unless it was already given.
- When output includes warnings or partial results, summarize them and ask how to adjust using `AskUserQuestion`.
- If phase boundaries or role ownership are violated, stop and request clarification.
