---
name: codex
description: Use when the user asks to run Codex CLI (codex exec, codex resume) or references OpenAI Codex for code analysis, refactoring, or automated editing
---

# Codex Skill Guide

## Collaboration Modes

This skill supports two collaboration modes between **Claude Code** and **Codex**. Choose the mode based on task complexity.

### Serial Mode (Default)

The classic three-phase workflow. Use for single-focus tasks where one agent works at a time.

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

### Parallel Worktree Mode

Use for large tasks that can be split into independent modules. Claude Code and Codex work **simultaneously** in separate git worktrees on different branches, then merge via an integration branch.

#### When to Use Parallel Mode
- Task is naturally splittable into 2+ independent modules/files.
- No tight coupling between the subtasks (minimal shared files).
- Time savings from parallelism outweigh the merge overhead.

#### Parallel Workflow

```
1. Plan & Split     Claude Code analyzes the task, splits into subtasks,
                    assigns file ownership per agent.

2. Bootstrap        Create branches and worktree:
                    - cc/<task>  → Claude Code (main worktree)
                    - cx/<task>  → Codex (separate worktree)

3. Parallel Execute Both agents work simultaneously in their own worktrees.
                    Each agent only touches files it owns.

4. Integrate        Merge both branches into int/<task>, run tests,
                    resolve any conflicts.

5. Accept           Review the integrated result, merge to main.
```

#### Bootstrap Commands

```bash
# From the project root (main worktree)
BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@')
git switch "$BASE_BRANCH" && git pull

# Create branches (use -f to reset if they already exist from a previous attempt)
git branch -f cc/<task>
git branch -f cx/<task>
git worktree add ../<project>-codex cx/<task> 2>/dev/null || \
  echo "Worktree already exists; reusing ../<project>-codex"
git switch cc/<task>

# Ensure .coord/ is gitignored (per-repo, without modifying .gitignore)
mkdir -p .coord
grep -qxF '.coord/' .git/info/exclude 2>/dev/null || echo '.coord/' >> .git/info/exclude

# Launch Codex in its worktree
codex exec -C ../<project>-codex --skip-git-repo-check \
  -m gpt-5.3-codex --config model_reasoning_effort="high" \
  --sandbox workspace-write --full-auto \
  "<task package prompt>" 2>/dev/null
```

#### Integration Commands

```bash
# After both agents complete their work
BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@')

# Check for file overlap before merging
echo "=== Overlap check ==="
comm -12 \
  <(git diff --name-only "$BASE_BRANCH"...cc/<task> | sort) \
  <(git diff --name-only "$BASE_BRANCH"...cx/<task> | sort)
# If any files are listed above, resolve ownership before proceeding.

git switch -c int/<task> "$BASE_BRANCH"
git merge --no-ff cc/<task>
git merge --no-ff cx/<task>
# Run tests, resolve conflicts if any
# If clean: merge int/<task> into $BASE_BRANCH
```

#### File Ownership Rules
- Each file/directory is assigned to **exactly one agent** during the Plan phase.
- Shared files (e.g., routes, schemas, lockfiles, config) must be flagged as **"serial handoff"** — only one agent may edit them at a time, with an explicit handoff.
- If an agent discovers it needs to modify a file owned by the other, it must **stop and signal** rather than edit directly.

#### Worktree Session Management
- In parallel mode, prefer explicit session IDs over `resume --last` to avoid cross-worktree ambiguity.
- After launching Codex, record the session ID in `.coord/sessions.json`:
  ```bash
  # Capture session ID from Codex output (look for session_id in output)
  # Then record it:
  echo '{"codex_worktree": "../<project>-codex", "session_id": "<SESSION_ID>"}' > .coord/sessions.json
  ```
- Resume Codex in its worktree using the recorded session ID:
  ```bash
  echo "continue with XYZ" | codex exec -C ../<project>-codex \
    --skip-git-repo-check resume <SESSION_ID> - 2>/dev/null
  ```

#### Coordination Protocol (`.coord/`)

For parallel mode, use a `.coord/` directory in the project root to coordinate state between agents. This directory is gitignored via `.git/info/exclude` during bootstrap (see Bootstrap Commands).

| File | Purpose |
| --- | --- |
| `.coord/plan.yaml` | Task split, subtask assignments, file ownership |
| `.coord/claims.yaml` | Active file/module claims per agent |
| `.coord/sessions.json` | Worktree path → Codex session ID mapping |
| `.coord/events.jsonl` | Append-only event log (handoff, blocked, done) |

**Event format:**
```json
{"ts":"2026-02-07T10:00:00Z","from":"claude-code","type":"done","subtask":"auth-module","files":["src/auth.ts","src/auth.test.ts"]}
{"ts":"2026-02-07T10:05:00Z","from":"codex","type":"blocked","subtask":"api-routes","need":"schema decision from claude-code"}
```

**Event types:** `started`, `done`, `blocked`, `handoff`, `claim`, `release`

Claude Code should check `.coord/events.jsonl` after Codex completes to understand its status and any blockers.

#### Integration Gate

Before merging into `int/<task>`:
1. Run the overlap check from Integration Commands. If any files appear in both diffs, stop and resolve ownership before merging.
2. If overlap is empty, proceed with the merge.
3. Run the full test suite on the integration branch.
4. Verify all acceptance criteria from both subtask packages.

#### Cleanup

After successful integration:
```bash
git worktree remove ../<project>-codex
git branch -d cc/<task> cx/<task> int/<task>
rm -rf .coord/
```

## Task Package Format (Standard Prompt Template)
Codex must receive a normalized Task Package. If missing, ask Claude Code to supply it.

```
[Task Package]
Goal:
- <single-sentence goal>

Mode: serial | parallel
Branch: <branch name, required for parallel mode>
Worktree: <worktree path, required for parallel mode>

Files:
- <explicit file list>

Ownership:
- <agent>: <file globs this agent may edit>
- DO NOT TOUCH: <file globs reserved for the other agent>

Constraints:
- <non-functional requirements, style rules, time limits, deps>

Acceptance Criteria:
- <observable outcomes>
- <tests to run or checks to pass>

Notes:
- <assumptions, risk areas, or context snapshots>
```

For serial mode, `Mode`, `Branch`, `Worktree`, and `Ownership` fields may be omitted.

## Running a Task
1. Confirm the Task Package exists and is complete.
2. Ask the user (via `AskUserQuestion`) which model to run (`gpt-5.3-codex` or `gpt-5.3`) AND which reasoning effort to use (`xhigh`, `high`, `medium`, or `low`) in a **single prompt with two questions**.
3. Select the sandbox mode required for the task; default to `--sandbox read-only` unless edits or network access are necessary.
4. Assemble the command with the appropriate options:
   - `-m, --model <MODEL>`
   - `--config model_reasoning_effort="<xhigh|high|medium|low>"`
   - `--sandbox <read-only|workspace-write|danger-full-access>`
   - `--full-auto`
   - `-C, --cd <DIR>` (use worktree path for parallel mode)
   - `--skip-git-repo-check`
5. Always use `--skip-git-repo-check`.
6. When continuing a previous session, use `codex exec --skip-git-repo-check resume --last` via stdin. When resuming, don't use any configuration flags unless explicitly requested by the user (e.g., model or reasoning effort). Resume syntax:
   - `echo "your prompt here" | codex exec --skip-git-repo-check resume --last - 2>/dev/null`
   - All flags have to be inserted between `exec` and `resume`.
   - **Parallel mode**: prefer `resume <SESSION_ID>` over `resume --last` to avoid cross-worktree ambiguity.
7. **IMPORTANT**: By default, append `2>/dev/null` to all `codex exec` commands to suppress thinking tokens (stderr). Only show stderr if the user explicitly requests to see thinking tokens or if debugging is needed. **On non-zero exit**, re-run without `2>/dev/null` to capture error details before reporting.
8. Run the command, capture stdout/stderr (filtered as appropriate), and summarize the outcome for the user.
9. **After Codex completes**, inform the user: "You can resume this Codex session at any time by saying 'codex resume' or asking me to continue with additional analysis or changes."

## Smart Sandbox Selection
Choose the minimum required access. Prefer least privilege.

| Use case | Sandbox mode | Key flags |
| --- | --- | --- |
| Read-only review or analysis | `read-only` | `--sandbox read-only 2>/dev/null` |
| Apply local edits | `workspace-write` | `--sandbox workspace-write --full-auto 2>/dev/null` |
| Permit network or broad access | `danger-full-access` | `--sandbox danger-full-access 2>/dev/null` |
| Resume recent session | Inherited from original | `echo "prompt" \| codex exec --skip-git-repo-check resume --last 2>/dev/null` |
| Run from another directory | Match task needs | `-C <DIR>` plus other flags `2>/dev/null` |
| Parallel worktree | `workspace-write` | `-C ../<project>-codex --sandbox workspace-write --full-auto 2>/dev/null` |

If unsure, default to `read-only` and ask if writes/network are needed.

## Acceptance Workflow (Claude Code)
After Codex execution, Claude Code should:

1. Perform diff review of modified files.
2. Run tests or checks listed in `Acceptance Criteria`.
3. Verify constraints and confirm no files outside the `Ownership` / `Files` boundary were modified.
4. For parallel mode: check `.coord/events.jsonl` for blockers or handoff requests.
5. Decide one of:
   - **Accept** — proceed to next task or commit (or integration for parallel mode)
   - **Revise** — re-enter Execute phase with feedback
   - **Abort** — revert changes

## Conflict Avoidance

### Serial Mode: Role Locking

- **Explore phase**: Claude Code is read-only. Codex must not write.
- **Execute phase**: Codex writes; Claude Code does not change files.
- **Accept phase**: Claude Code reviews and validates; Codex does not write unless explicitly re-entering Execute phase.

### Parallel Mode: File Ownership Locking

- Each file/module is assigned to one agent in the Task Package `Ownership` field.
- Both agents may write simultaneously, but **only to their owned files**.
- Shared files require explicit **claim/release** via `.coord/claims.yaml`.
- Before editing a shared file, check that no other agent holds a claim on it.

If a phase or ownership boundary is unclear, stop and request clarification.

## Following Up
- If the task package is incomplete, request clarification **before** running Codex.
- After every `codex` command, use `AskUserQuestion` to confirm next steps, collect clarifications, or decide whether to resume with `codex exec resume --last`.
- Restate the chosen model, reasoning effort, and sandbox mode when proposing follow-up actions.

## Error Handling
- Stop and report failures whenever `codex --version` or a `codex exec` command exits non-zero; request direction before retrying.
- Before you use high-impact flags (`--full-auto`, `--sandbox danger-full-access`, `--skip-git-repo-check`) ask the user for permission using AskUserQuestion unless it was already given.
- When output includes warnings or partial results, summarize them and ask how to adjust using `AskUserQuestion`.
- If phase boundaries or role ownership are violated, stop and request clarification.
- **Parallel mode**: if worktree creation fails, fall back to serial mode and inform the user.
