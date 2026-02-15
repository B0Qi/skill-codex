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

# Launch Codex in its worktree (ALWAYS verify and use absolute path for -C)
WORKTREE="$(cd ../<project>-codex && pwd)"
test -d "$WORKTREE" || { echo "ERROR: worktree not found at $WORKTREE"; exit 1; }
CX="$HOME/.claude/skills/codex/scripts/cx-parse"
JSONL="$(mktemp -t codex.XXXXXX.jsonl)"
PROMPT="<task package prompt>"
codex exec --json -C "$WORKTREE" --skip-git-repo-check \
  -m gpt-5.3-codex -c model_reasoning_effort='"high"' \
  --yolo \
  "$PROMPT" >"$JSONL" 2>/dev/null
SESSION_ID="$($CX session-id "$JSONL")"
RESPONSE="$($CX response "$JSONL" --max-chars 12288)"
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
- Uses the shared **Session Registry** (`.coord/sessions.jsonl`). In parallel mode, each worktree's Codex session gets its own registry entry with `mode: parallel` and the worktree `cwd`.
- Resume Codex in its worktree using the recorded session ID. Note: `resume` does NOT support `-C` — you must `cd` to the worktree first:
  ```bash
  CX="$HOME/.claude/skills/codex/scripts/cx-parse"
  JSONL="$(mktemp -t codex.XXXXXX.jsonl)"
  WORKTREE="$(cd ../<project>-codex && pwd)"
  cd "$WORKTREE" && codex exec --json --yolo \
    --skip-git-repo-check resume <SESSION_ID> "<resume prompt>" >"$JSONL" 2>/dev/null
  SESSION_ID="$($CX session-id "$JSONL")"
  RESPONSE="$($CX response "$JSONL" --max-chars 12288)"
  ```

#### Coordination Protocol (`.coord/`)

For parallel mode, use a `.coord/` directory in the project root to coordinate state between agents. This directory is gitignored via `.git/info/exclude` during bootstrap (see Bootstrap Commands).

| File | Purpose |
| --- | --- |
| `.coord/plan.yaml` | Task split, subtask assignments, file ownership |
| `.coord/claims.yaml` | Active file/module claims per agent |
| `.coord/sessions.jsonl` | Session registry (shared across serial and parallel modes) |
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

## Session Registry

Codex sessions are persistent (`~/.codex/sessions/`) and can be resumed by ID at any time. To survive timeouts, context loss, and long-running tasks, Claude Code maintains a **session registry** — a lightweight mapping from session IDs to task descriptions.

### Registry File

Use `.coord/sessions.jsonl` (append-only, one JSON object per line). Initialize `.coord/` if it does not exist:

```bash
mkdir -p .coord
grep -qxF '.coord/' .git/info/exclude 2>/dev/null || echo '.coord/' >> .git/info/exclude
```

### Registry Entry Format

Append one line per agent session:

```json
{"agent":"codex","session_id":"<ID>","session_ref":{"type":"id","value":"<ID>"},"task":"<short task description>","status":"running","cwd":"<working dir>","mode":"serial","created_at":"<ISO8601>"}
```

**Fields:**

| Field | Required | Description |
| --- | --- | --- |
| `agent` | yes | Agent that owns this session: `codex` or `gemini`. Defaults to `"codex"` if omitted (backward-compatible). |
| `session_id` | yes | Unified session identifier (Codex session ID or a generated ID for Gemini) |
| `session_ref` | yes | Agent-specific session reference for resume. See below. |
| `task` | yes | One-line description of what this session is doing |
| `status` | yes | `running`, `paused`, `done`, `failed` |
| `cwd` | yes | Working directory (main project root or worktree path) |
| `mode` | yes | `serial` or `parallel` |
| `created_at` | yes | ISO 8601 timestamp |
| `branch` | parallel only | Branch name (`cx/<task>` or `gm/<task>`) |
| `updated_at` | no | Last status change timestamp |
| `notes` | no | Free-form context (blockers, partial results, etc.) |

**`session_ref` by agent:**

| Agent | `session_ref.type` | `session_ref.value` | Notes |
| --- | --- | --- | --- |
| codex | `id` | Codex session ID string | Stable — can resume directly by ID |
| gemini | `index` | Integer session index | **Volatile** — must re-resolve via `gemini --list-sessions` before resume, as indices shift |

Entries without `agent` or `session_ref` are treated as legacy Codex entries (`agent: "codex"`, `session_ref` derived from `session_id`).

### Registry Lifecycle

1. **On launch**: after running `codex exec`, extract the session ID and append a registry entry with `status: running`.
2. **On completion**: update the entry's status to `done`.
3. **On timeout/interrupt**: status stays `running` — on next interaction, Claude Code reads the registry, finds the active session, and resumes it.
4. **On failure**: update status to `failed`, add error details to `notes`.

### Extracting Session ID & Response

**Always use `--json` mode** to capture the session ID reliably. The `--json` flag outputs structured JSONL events to stdout, and the very first line is a `thread.started` event containing the session UUID:

```json
{"type":"thread.started","thread_id":"019c50f4-16ca-7711-afae-eecea2cb60a2"}
```

> **IMPORTANT — Context-window protection:** The `--json` JSONL stream contains ALL intermediate events (file reads, tool calls, command outputs) and can easily reach **50-100KB+** for multi-file tasks. **NEVER capture the full stream into a shell variable** (`OUTPUT=$(codex exec ...)`). Always redirect to a temp file and extract only the two fields you need: `thread_id` and the final `item.completed` response text.

**Parser shorthand** (available after `install.sh`):
```bash
CX="$HOME/.claude/skills/codex/scripts/cx-parse"
```

`cx-parse` is a zero-dependency Python 3 CLI that extracts `session_id` and `response` from Codex JSONL without loading the full stream into memory. Exit codes: 0=ok, 2=not-found/invalid, 3=parse-error. UUID validation is built in.

**Extraction pattern (foreground):**
```bash
CX="$HOME/.claude/skills/codex/scripts/cx-parse"
JSONL="$(mktemp -t codex.XXXXXX.jsonl)"

# Redirect JSONL to file — nothing enters Claude Code's context yet
codex exec --json [OPTIONS] "$PROMPT" >"$JSONL" 2>/dev/null

SESSION_ID="$($CX session-id "$JSONL")"
RESPONSE="$($CX response "$JSONL" --max-chars 12288)"
# $JSONL remains on disk for diagnostics; clean up when no longer needed
```

**Extraction pattern (background):**
```bash
CX="$HOME/.claude/skills/codex/scripts/cx-parse"
JSONL="$(mktemp -t codex.XXXXXX.jsonl)"
ERR="$(mktemp -t codex.XXXXXX.err)"

codex exec --json [OPTIONS] "$PROMPT" >"$JSONL" 2>"$ERR" &
CODEX_PID=$!

# Poll for session ID (reads only the first line, not the whole file)
SESSION_ID=""
for i in $(seq 1 120); do
  [ -s "$JSONL" ] && SESSION_ID="$($CX session-id "$JSONL" 2>/dev/null)" && [ -n "$SESSION_ID" ] && break
  sleep 0.5
done

# When checking status: NEVER read the full JSONL. Use targeted extraction:
#   tail -n 5 "$ERR"              # last few stderr lines for diagnostics
#   wc -l < "$JSONL"              # event count as progress indicator

# After completion, extract response:
RESPONSE="$($CX response "$JSONL" --max-chars 12288)"
```

**JSON extraction** (session_id + text + metadata in one call):
```bash
$CX extract "$JSONL" --max-chars 12288
# stdout: {"session_id":"...","text":"...","text_truncated":false,"metadata":{"source":"..."}}
```

**Pipe mode with `--tee`** (save stdin copy then parse):
```bash
codex exec --json [OPTIONS] "$PROMPT" 2>/dev/null | $CX extract --tee "$JSONL" --max-chars 12288
```

**Fallback (inline):** If `cx-parse` is unavailable, use inline Python:
```bash
SESSION_ID="$(python3 -c "
import json,sys
with open(sys.argv[1],'r',errors='replace') as f: obj=json.loads(f.readline())
print(obj.get('thread_id',''))
" "$JSONL")"

RESPONSE="$(python3 -c "
import json,sys; path,cap=sys.argv[1],int(sys.argv[2]); last=''
with open(path,'r',errors='replace') as f:
    for line in f:
        try: obj=json.loads(line)
        except: continue
        if obj.get('type')=='item.completed':
            item=obj.get('item') or {}
            if item.get('type')=='agent_message': last=item.get('text') or ''
if len(last)>cap: print(last[:cap]+f'\n[truncated {len(last)-cap} chars]')
else: print(last)
" "$JSONL" 12288)"
```

**Fallback (stderr grep):** If `--json` is unavailable, extract from stderr:
```bash
codex exec [OPTIONS] "$PROMPT" 2>/tmp/codex_stderr.txt
SESSION_ID=$(grep "session id:" /tmp/codex_stderr.txt | awk '{print $NF}')
```

> **WARNING:** The old `find ~/.codex/sessions -newer ... | jq` method is **deprecated** — it is unreliable and has produced garbage values in production. Do not use it.

### Context Recovery

When Claude Code loses context (conversation compression, new session, etc.):
1. Read `.coord/sessions.jsonl` to find sessions with `status: running`. **Use `tail -n 50`** — never read the entire file, as it is append-only and grows over time.
2. Filter by `agent` field — only act on entries matching the current skill context (e.g., `agent: "codex"` for this skill). Entries without an `agent` field default to `"codex"`.
3. Read the original Task Package (if saved in `.coord/` or conversation).
4. Use `session_ref` to construct the correct resume command for the agent type.
5. Resume the session with a status-check prompt (see Resume Protocol below).

> **Append-only log hygiene:** `.coord/sessions.jsonl` and `.coord/events.jsonl` are append-only and can grow indefinitely. Always use `tail -n N` or filter by status/timestamp when reading them. Never `cat` the entire file into context.

## Resume Protocol

When resuming a Codex session (whether after timeout, context loss, or user-initiated), always use this protocol:

1. **Resume with a self-check prompt.** The first message to a resumed session must ask Codex to summarize its current state:
   ```
   Resume. Before continuing, output a brief status summary:
   1. What has been completed so far
   2. What files were modified
   3. What remains to be done
   4. Any blockers or decisions needed
   Then continue with the remaining work.
   ```
2. **Use `session_ref` to construct the resume command.** Look up the registry entry and use `session_ref.type` to determine the correct command:
   - **Codex** (`session_ref.type: "id"`): use the session ID directly:
     ```bash
     codex exec --json --yolo --skip-git-repo-check resume <session_ref.value> "<resume prompt>" 2>/dev/null
     ```
   - **Gemini** (`session_ref.type: "index"`): re-resolve the index first, then resume (see `/gemini` skill for details).
   - **Fallback**: if `session_ref` is missing (legacy entry), use `session_id` as a Codex session ID.
3. **CWD matters for resume.** `codex exec resume` filters sessions by current working directory by default.
   - **Best practice**: `cd` to the `cwd` recorded in the registry entry before resuming.
   - **Fallback**: use `--all` to bypass CWD filtering: `codex exec --json --yolo --skip-git-repo-check resume --all <SESSION_ID> "<prompt>" 2>/dev/null`
   - **Note**: `codex exec resume` does NOT support `-C` — you must physically `cd` to the correct directory.
   - When outputting resume commands for the user, always include the `cd` prefix:
     ```bash
     cd /path/to/original/cwd && codex exec --json --yolo --skip-git-repo-check resume <SESSION_ID> "<prompt>" 2>/dev/null
     ```
4. **After timeout or error**: temporarily remove `2>/dev/null` on the next resume to capture any diagnostic output, then re-add it once confirmed working.
5. **Update the registry** after resume completes (status, updated_at, notes).
6. **Resume count tracking.** Each resume adds context to the conversation. To prevent cumulative context bloat:
   - Track resume count in the registry entry's `notes` field (e.g., `"notes":"resume #3"`).
   - After **3 resumes** of the same session, perform a "checkpoint": summarize all prior work into a short bullet list and save it to `.coord/<session_id>-summary.md`. On the next resume, use only this summary as context instead of carrying forward raw outputs.
   - After **5 resumes**, recommend the user start a fresh session with a new Task Package that incorporates completed work.

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

## Available Models

| Model | Description | Best For |
| --- | --- | --- |
| `gpt-5.3-codex` | Latest frontier agentic coding model (default) | Complex multi-file tasks, architecture changes |
| `gpt-5.3-codex-spark` | Ultra-fast coding model | Quick edits, simple tasks, rapid iteration |
| `gpt-5.2-codex` | Frontier agentic coding model (previous gen) | Stable fallback if 5.3 has issues |
| `gpt-5.1-codex-max` | Codex-optimized flagship for deep and fast reasoning | Deep analysis, complex debugging |
| `gpt-5.2` | Latest frontier model with broad improvements | General-purpose, knowledge-heavy tasks |
| `gpt-5.1-codex-mini` | Optimized for codex, cheaper and faster | Cost-sensitive tasks, simple fixes |

## Running a Task
1. Confirm the Task Package exists and is complete.
2. Ask the user (via `AskUserQuestion`) which model to run (default: `gpt-5.3-codex`) AND which reasoning effort to use (`xhigh`, `high`, `medium`, or `low`) in a **single prompt with two questions**. Refer to the **Available Models** table above when presenting options.
3. Select the permissions mode (see **Permissions Model** section below). Default is `--yolo` (full access). Use safe mode only when the user explicitly requests it or the task involves an untrusted codebase.
4. Assemble the command with the appropriate options:
   - `--json` (REQUIRED — enables structured session ID extraction on stdout)
   - `-m, --model <MODEL>`
   - `-c model_reasoning_effort='"<xhigh|high|medium|low>"'` (note: value must be TOML — wrap strings in single-quote + double-quote)
   - `--yolo` (default) or `--sandbox workspace-write --full-auto -c sandbox_workspace_write.network_access=true` (safe mode)
   - `-C, --cd <DIR>` (use worktree path for parallel mode)
   - `--skip-git-repo-check`
5. Always use `--skip-git-repo-check`.
6. **Prompt passing strategy** (IMPORTANT — `-p` flag is unreliable for multiline prompts):
   - **Short prompts** (single line, no special chars): pass directly as positional argument: `codex exec [OPTIONS] "prompt text"`
   - **Long/multiline prompts** (Task Packages): write the prompt to a temp file, then pass via shell variable:
     ```bash
     PROMPT=$(cat /path/to/prompt_file.txt)
     codex exec [OPTIONS] "$PROMPT" 2>/dev/null
     ```
   - **NEVER use `-p` with multiline text** — Codex CLI may misparse it as a config profile name (especially if prompt starts with `[`).
   - **NEVER use trailing `-`** for stdin — it is not a valid positional argument for `codex exec`.
7. When continuing a previous session, look up the session ID from `.coord/sessions.jsonl` and use `resume <SESSION_ID>`. Follow the Resume Protocol (self-check prompt first). Resume syntax:
   ```bash
   codex exec --json --yolo --skip-git-repo-check resume <SESSION_ID> "Resume prompt here" 2>/dev/null
   ```
   - **`--json` and `--yolo` are REQUIRED on resume** — permissions and output mode do not carry over from the original session. Without `--yolo`, resume falls back to default sandbox mode.
   - All flags must be inserted between `exec` and `resume`.
   - Model/reasoning flags are not needed on resume (inherited from original session).
   - Fall back to `resume --last` only if no registry entry exists.
8. **IMPORTANT — stderr handling**:
   - By default, append `2>/dev/null` to foreground `codex exec` commands to suppress thinking tokens (stderr). With `--json`, agent output goes to stdout as structured JSONL, so `2>/dev/null` is safe and recommended.
   - **In background mode**, redirect stdout and stderr to **separate files** (`>"$JSONL" 2>"$ERR"`). Do NOT mix them with `2>&1` — mixing corrupts the JSONL stream and inflates the output with thinking tokens. See **Extraction pattern (background)** for the correct pattern.
   - **On non-zero exit** of foreground commands, re-run without `2>/dev/null` to capture error details before reporting.
9. **IMPORTANT — path validation for `-C` flag**:
   - Always use **absolute paths** with `-C`, never relative paths like `../<project>-codex`.
   - Before launching Codex, **verify the `-C` path exists**: `ls -d /path/to/worktree` or `test -d /path/to/worktree`.
   - Common mistake: nested project structures (e.g., `casimir2/casimir/`) cause `../project-codex` to resolve to wrong level.
10. Run the command using the **Extraction patterns** from the "Extracting Session ID & Response" section. **NEVER capture raw JSONL into a shell variable** — always redirect to a temp file and extract only `SESSION_ID` + capped `RESPONSE`. Summarize the outcome for the user.
11. **After launching Codex**, extract the session ID from the temp file (see "Extracting Session ID & Response") and append a registry entry to `.coord/sessions.jsonl` (see Session Registry). **Validate** the ID is UUID format before recording.
12. **After Codex completes**, update the registry entry status to `done` and inform the user: "You can resume this Codex session at any time by saying 'codex resume' or asking me to continue with additional analysis or changes."

## Permissions Model

Codex uses a **2-tier permissions model** to match Claude Code's own access level and avoid sandbox-related failures in multi-agent workflows.

### Default: Full Access (`--yolo`)

Always use `--dangerously-bypass-approvals-and-sandbox` (alias `--yolo`) unless the user explicitly requests safe mode. This flag:
- Disables the OS-level sandbox (Landlock+seccomp on Linux, Seatbelt on macOS)
- Skips all approval prompts
- Grants full filesystem, network, and process access — matching Claude Code's own permissions

**Rationale:** Claude Code already runs unsandboxed. Sandboxing only Codex creates a false security boundary while introducing complexity and failure modes (blocked sockets, write denials, network timeouts) that compound in multi-agent workflows.

```bash
# Standard command pattern (default)
codex exec --yolo --skip-git-repo-check \
  -m gpt-5.3-codex -c model_reasoning_effort='"high"' \
  "$PROMPT" 2>/dev/null
```

### Safe Mode: Sandboxed with Network (opt-in)

Use when working with **untrusted codebases** or when the user explicitly requests sandboxing. This mode restricts filesystem writes to the workspace while still allowing network access:

```bash
# Safe mode command pattern
codex exec --skip-git-repo-check \
  --sandbox workspace-write --full-auto \
  -c sandbox_workspace_write.network_access=true \
  -m gpt-5.3-codex -c model_reasoning_effort='"high"' \
  "$PROMPT" 2>/dev/null
```

### When to Use Each Mode

| Scenario | Mode | Flag |
| --- | --- | --- |
| Own codebase, local dev (default) | Full access | `--yolo` |
| Reviewing untrusted PR or repo | Safe mode | `--sandbox workspace-write --full-auto -c sandbox_workspace_write.network_access=true` |
| User explicitly asks for sandbox | Safe mode | (same as above) |
| Parallel worktree | Full access | `--yolo -C /absolute/path/to/worktree` |
| Resume session | Full access | `--json --yolo ... resume <SESSION_ID>` (permissions NOT inherited) |

## Acceptance Workflow (Claude Code)
After Codex execution, Claude Code should:

1. **Size-aware diff review.** Use a tiered strategy to prevent large diffs from flooding the context window:
   - Always start with `git diff --stat` and `git diff --name-only` for an overview.
   - **≤10 files changed**: review full diffs with `git diff --unified=3 -- <file>` per file.
   - **>10 files changed**: review only `--stat` + `--name-only`. For spot-checks, pick 3-5 representative files and show their diffs individually (limit to first 200 lines each with `| head -200`).
   - **Always exclude generated/lock files** from full diff: `git diff -- ':!package-lock.json' ':!pnpm-lock.yaml' ':!*.lock'`
2. Run tests or checks listed in `Acceptance Criteria`.
3. Verify constraints and confirm no files outside the `Ownership` / `Files` boundary were modified.
4. For parallel mode: check `.coord/events.jsonl` (**use `tail -n 50`**, not full read) for blockers or handoff requests.
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
- After every `codex` command, update `.coord/sessions.jsonl` and use `AskUserQuestion` to confirm next steps, collect clarifications, or decide whether to resume with `codex exec resume <SESSION_ID>`.
- Restate the chosen model and reasoning effort when proposing follow-up actions.
- If conversation context has been lost, read `.coord/sessions.jsonl` to recover active session state before taking action.

## Error Handling
- Stop and report failures whenever `codex --version` or a `codex exec` command exits non-zero; request direction before retrying.
- The `--yolo` flag is the default for this skill and does not require additional user confirmation. If the user switches to safe mode, confirm before reverting to `--yolo`.
- When output includes warnings or partial results, summarize them and ask how to adjust using `AskUserQuestion`.
- If phase boundaries or role ownership are violated, stop and request clarification.
- **Parallel mode**: if worktree creation fails, fall back to serial mode and inform the user.
- **Timeout recovery**: if a `codex exec` command is interrupted by timeout, the session is still recoverable. Look up the session ID from `.coord/sessions.jsonl`, then resume using the Resume Protocol. On the first resume after a timeout, temporarily remove `2>/dev/null` to capture any diagnostic stderr.
