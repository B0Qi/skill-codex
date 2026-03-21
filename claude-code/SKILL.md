---
name: codex
description: Use when the user asks to run Codex CLI (codex exec, codex resume) or references OpenAI Codex for code analysis, refactoring, or automated editing. Also trigger when the user wants to delegate a coding task to Codex, mentions running tasks with GPT models, wants parallel agent execution in worktrees, or says things like "have codex do it", "send this to codex", or "let another agent handle this".
---

# Codex Skill Guide

## Delegation Discipline

The entire purpose of this skill is to delegate work to Codex and let it finish autonomously. Claude Code's role after launch is to **wait** — not to help, not to "speed things up", not to take over. Long-running tasks are normal and expected; a task taking 5–10 minutes (or longer) is not a sign that something is wrong.

**Rules that must never be broken:**

1. **Never kill a Codex process.** Do not run `kill`, `pkill`, `killall`, or any signal against a Codex PID. Do not cancel a background task that is running Codex. If you feel the urge to terminate Codex and do the work yourself — that impulse is wrong. Resist it.
2. **Never take over delegated work.** Once a task has been sent to Codex, that task belongs to Codex until it reaches a verified terminal state (completion event or confirmed process death). A watcher timeout is NOT a terminal state — it only means the watcher stopped waiting, not that Codex stopped working. Do not start editing the same files, running the same commands, or reimplementing the same logic. Even if you "know" how to do it faster — the user chose to delegate, so respect that choice.
3. **Never preemptively intervene.** Do not check on Codex "just to see how it's going" and then decide to take over. Do not interpret silence or slow progress as failure. Wait for the background task notification.
4. **After launch, stop working on that task.** Tell the user Codex is running, then either wait for the notification or work on something unrelated if the user asks. Do not fill the waiting time by doing the delegated task yourself.

The value of delegation is that Codex works independently. Every time CC kills a Codex process or takes over mid-task, it wastes the tokens Codex already spent, confuses the file state, and defeats the purpose of having two agents. The correct action when a long task is running is: do nothing and wait.

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

For large tasks splittable into independent modules. Claude Code and Codex work simultaneously in separate git worktrees, then merge via an integration branch.

> **Full instructions:** Read `references/parallel-mode.md` before using parallel mode. It covers bootstrap, file ownership, coordination protocol, integration gate, and cleanup.

## Session Registry

Codex sessions are persistent (`~/.codex/sessions/`) and can be resumed by ID at any time. Claude Code maintains a **session registry** (`.coord/sessions.jsonl`) — a lightweight append-only mapping from session IDs to task descriptions, enabling recovery from timeouts and context loss.

Initialize `.coord/` if it does not exist:

```bash
mkdir -p .coord
grep -qxF '.coord/' .git/info/exclude 2>/dev/null || echo '.coord/' >> .git/info/exclude
```

### Registry Entry Format

Append one line per agent session:

```json
{"agent":"codex","coord_id":"<COORD_ID>","session_id":"<ID>","session_ref":{"type":"id","value":"<ID>"},"task":"<short task description>","status":"running","cwd":"<working dir>","mode":"serial","created_at":"<ISO8601>"}
```

**Key fields:** `agent` (codex/gemini), `coord_id` (Claude-generated task identifier — see Task State Synchronization), `session_id`, `session_ref` (agent-specific resume reference), `task`, `status` (running/paused/done/failed), `cwd`, `mode` (serial/parallel), `created_at`. Parallel mode also uses `branch`.

**`session_ref` by agent:**

| Agent | `session_ref.type` | `session_ref.value` | Notes |
| --- | --- | --- | --- |
| codex | `id` | Codex session ID string | Stable — can resume directly by ID |
| gemini | `index` | Integer session index | Volatile — re-resolve via `gemini --list-sessions` before resume |

Entries without `agent` or `session_ref` are treated as legacy Codex entries.

### Registry Lifecycle

1. **On launch**: extract session ID, append entry with `status: running`.
2. **On completion**: update status to `done`.
3. **On watch timeout** (rare — timeout is disabled by default): status stays `running` — Codex process may still be alive. Reattach or wait, do NOT take over.
4. **On failure**: update status to `failed`, add error details to `notes`.

> **Append-only log hygiene:** Always use `tail -n 50` when reading `.coord/sessions.jsonl` and `.coord/events.jsonl`. Never `cat` the entire file into context.

## Task State Synchronization

During long-running tasks, Claude Code may lose track of an active Codex session — due to bash timeouts, context compression, or new user interactions. Without a guard, Claude Code starts editing files that Codex is still working on, causing duplicated effort or conflicts.

This is solved at the **CLI infrastructure level** — `cx-parse watch` monitors the Codex process and JSONL stream, manages `.coord/` state files automatically, and notifies Claude Code on completion. No token waste, no model-side heartbeats.

### Coord ID

Each task gets a `coord_id` generated by Claude Code before launch:

```bash
COORD_ID="cx_$(date +%s)"
```

### How It Works

`cx-parse watch` runs alongside the Codex process and handles the full lifecycle:

1. **Monitors** the JSONL file for `task_complete` events (CLI-level signal, not model-dependent)
2. **Tracks** the Codex PID — detects crashes even if no terminal event is written
3. **Writes heartbeat** to `.coord/running/<coord_id>.json` every 2 seconds while Codex is alive
4. **On completion**: writes `.coord/done/<coord_id>.json`, removes running file, outputs result JSON
5. **On crash**: writes `.coord/failed/<coord_id>.json`, removes running file
6. **On watch timeout** (only if `--timeout` is explicitly set): keeps running file, outputs `watch_timeout` status. Codex process is still alive — this is a watcher limit, not a task failure

Claude Code runs the whole thing with `run_in_background` and gets notified automatically — zero polling, zero wasted tokens.

### Launch Pattern

Use `cx-run` — a single command that wraps the entire lifecycle. Run it with `run_in_background`:

```bash
$HOME/.claude/skills/codex/scripts/cx-run -- \
  codex exec --json --yolo --skip-git-repo-check \
  -m gpt-5.4 -c model_reasoning_effort='"high"' \
  "$PROMPT"
```

By default, `cx-run` watches indefinitely — it waits as long as the Codex process is alive and only exits on completion or crash. Use `--timeout N` only if you need an explicit hard ceiling (rare).

`cx-run` handles everything internally: creates temp JSONL, launches Codex in background, extracts session ID, registers in `.coord/sessions.jsonl`, starts `cx-parse watch` to monitor PID + JSONL stream, manages `.coord/running/done/failed/` state files, and outputs result JSON when done.

**Output** (JSON to stdout on completion):
```json
{"status":"completed","session_id":"019c...","response":"...","response_truncated":false,"output_tokens":1234,"elapsed_seconds":45.2,"token_usage":{...}}
```

Key fields:
- `status`: `completed`, `crashed`, `watch_timeout` (watcher limit, not task failure — Codex may still be alive), or `error`
- `response_truncated`: `true` when response was capped by `--max-chars` (context protection — full response remains in the JSONL file)
- `output_tokens`: model's raw output token count — use this to detect model-level truncation (see Truncation Handling)

### Guard Check (Claude Code)

**Before any file edit in the same project**, Claude Code MUST:

1. Check `.coord/running/` for any `.json` files.
2. If an active entry exists (heartbeat `last_active` is fresh):
   - **Do NOT edit files.** Codex is still working.
   - **Do NOT kill the Codex process.** Do not run `kill`, `pkill`, or any signal against it.
   - **Do NOT start doing the delegated work yourself.** This is the most common mistake — CC sees an active task and decides to "help" by doing it in parallel. Don't.
   - Inform the user that Codex is still running, and wait for the background task notification.
3. If heartbeat is stale (not updated recently) but no matching `done/` or `failed/` file exists: the Codex process may still be alive but the watcher stopped. **Verify PID liveness** before proceeding — check the `pid` field in the running file and run `kill -0 <pid>` to test. If the PID is alive, do NOT edit files.
4. Only proceed normally if: (a) no running entries exist, or (b) a matching `done/` or `failed/` file confirms the task ended.

This is a **task-level guard** in serial mode. In parallel mode, the file ownership rules in `references/parallel-mode.md` apply instead.

### Edge Cases

- **Crash**: `cx-parse watch` detects PID death, writes `failed/<coord_id>.json`, removes running file. Claude Code reads the failure and decides to resume or ask the user — never silently take over.
- **Watch timeout** (only if `--timeout` was explicitly set): Watch exits with `watch_timeout` status but keeps the running file — Codex process is still alive. Do NOT take over. Either reattach a new watcher or wait for the user to decide.
- **Context loss**: Claude Code scans `.coord/running/` and `.coord/sessions.jsonl` to discover in-flight tasks.
- **Resume**: Generate a new `coord_id` for the resumed session's watcher.
- **Truncation**: See Truncation Handling below.

### Truncation Handling

When Codex hits the model's max output tokens, the response is cut off mid-stream. The JSONL still emits `turn.completed` so `status` is `"completed"`, but the response is incomplete. Detect this and resume.

**Detection — check `output_tokens` in the result JSON:**

| `output_tokens` | Likely state | Action |
| --- | --- | --- |
| < 30,000 | Normal completion | Proceed to Accept |
| ≥ 30,000 | Likely model-truncated | Check response, resume if incomplete |

> These thresholds are approximate. When in doubt, check if the response ends mid-sentence or mid-code-block.

**Resume on truncation** — use `cx-run --resume`:

```bash
$HOME/.claude/skills/codex/scripts/cx-run --resume <SESSION_ID> -- \
```

`cx-run --resume` automatically: inherits `--json --yolo --skip-git-repo-check`, sends a "Continue from where you left off" prompt, registers in the session registry with `resumed_from`, and watches to completion.

**Prevention — add a checkpoint hint to Task Packages for long-output tasks:**

When the task is expected to produce long output (code generation, reviews, reports), add this to the Task Package `Constraints`:

```
- If output is getting long, pause at a natural checkpoint and summarize progress so far before continuing.
```

This gives the model a chance to produce a complete intermediate result rather than being cut off mid-thought.

## Resume Protocol

When resuming a Codex session (after crash, truncation, context loss, or user request):

### Quick Resume (preferred)

Use `cx-run --resume` — handles everything automatically:

```bash
$HOME/.claude/skills/codex/scripts/cx-run --resume <SESSION_ID>
```

This inherits `--json --yolo --skip-git-repo-check`, sends a "Continue from where you left off" prompt, registers the resume in the session registry, and watches to completion. Run with `run_in_background` just like the original launch.

### Manual Resume

When you need a custom resume prompt or more control:

1. **Resume with a self-check prompt:**
   ```
   Resume. Before continuing, output a brief status summary:
   1. What has been completed so far
   2. What files were modified
   3. What remains to be done
   4. Any blockers or decisions needed
   Then continue with the remaining work.
   ```

2. **Construct the resume command** from `session_ref` in the registry:
   ```bash
   codex exec resume <session_ref.value> "Resume prompt here" --json --yolo --skip-git-repo-check 2>/dev/null
   ```
   - Syntax: `codex exec resume <SID> "prompt" [flags]` — the `resume` subcommand comes right after `exec`, then session ID, then prompt, then flags.
   - `--json` and `--yolo` are REQUIRED on resume — permissions and output mode do not carry over from the original session.
   - **CWD matters**: `cd` to the recorded `cwd` before resuming. `resume` does NOT support `-C`.
   - Fallback: use `--all` to bypass CWD filtering: `resume --all <SESSION_ID>`.
   - When outputting resume commands for the user, always include the `cd` prefix.

3. **After crash/error**: temporarily remove `2>/dev/null` to capture diagnostics.

4. **Update the registry** after resume completes (status, updated_at, notes).

### Resume Limits

Track resume count in `notes` field (e.g., `"notes":"resume #3"`). After **3 resumes**, checkpoint to `.coord/<session_id>-summary.md`. After **5 resumes**, recommend a fresh session with a new Task Package.

### Context Recovery

When Claude Code loses context:
1. Scan `.coord/running/` for active state files — this is the fastest way to detect in-flight tasks.
2. Cross-reference with `.coord/sessions.jsonl` (`tail -n 50`) to get session IDs for resume.
3. Filter by `agent` field (default `"codex"` if missing).
4. If a `done/` or `failed/` file exists for the coord_id, the task already finished — read the result instead of resuming.
5. Otherwise, use `session_ref` to construct the resume command and resume with the self-check prompt above.

## Task Package Format

Codex must receive a normalized Task Package. If missing, ask Claude Code to supply it.

```
[Task Package]
Coord-ID: <coord_id>

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

For serial mode, `Mode`, `Branch`, `Worktree`, and `Ownership` fields may be omitted. `Coord-ID` should always be included — `cx-parse watch` uses it for state file management.

## Available Models

Run `codex --help` or check OpenAI documentation for the latest model list. Common models as of last update:

| Model | Best For |
| --- | --- |
| `gpt-5.4` | Complex multi-file tasks, architecture (default) |
| `gpt-5.3-codex` | Stable fallback, code-heavy tasks |
| `gpt-5.3-codex-spark` | Quick edits, rapid iteration |
| `gpt-5.1-codex-max` | Deep analysis, complex debugging |
| `gpt-5.1-codex-mini` | Cost-sensitive, simple fixes |

## Running a Task

1. **Guard check**: scan `.coord/running/` for active entries. If any exist, do NOT proceed — wait for the notification or resume the existing session.
2. Confirm the Task Package is complete (include `Coord-ID`).
3. Ask the user which model (default: `gpt-5.4`) and reasoning effort (`xhigh`/`high`/`medium`/`low`) in a **single prompt**.
4. **Prompt passing**: for long/multiline prompts (Task Packages), write to a temp file first: `PROMPT=$(cat /path/to/task_package.txt)`. NEVER use `-p` with multiline text.
5. **Launch with `cx-run`** — a single `run_in_background` command that handles the entire lifecycle (launch, session registry, watch, state files):
   ```bash
   $HOME/.claude/skills/codex/scripts/cx-run -- \
     codex exec --json --yolo --skip-git-repo-check \
     -m gpt-5.4 -c model_reasoning_effort='"high"' \
     "$PROMPT"
   ```
   By default `cx-run` watches indefinitely until Codex completes or crashes. Do NOT add `--timeout` unless the user explicitly requests a hard ceiling.
   `cx-run` internally: launches Codex → extracts session ID → registers in `.coord/sessions.jsonl` → monitors via `cx-parse watch` → manages `.coord/running/done/failed/` → outputs result JSON → updates registry.
6. **Tell the user** Codex is running, then **stop completely**. Do NOT poll, check status, edit files, or work on the delegated task in any way. Do NOT kill the Codex process — even if it seems slow, even if you think you could do it faster, even if the user asks a follow-up question about the task. Wait for the background task notification. Long-running tasks (5–15+ minutes) are normal for complex work. If the user asks about progress, tell them Codex is still running — do not intervene.
7. **When notified**, read the result JSON and decide next action:
   - `status: completed` + `output_tokens < 30000` → proceed to Accept phase.
   - `status: completed` + `output_tokens ≥ 30000` → check if response ends mid-sentence/mid-code. If truncated, auto-resume with `cx-run --resume <session_id>`.
   - `status: crashed` (PID confirmed dead) → attempt resume via `cx-run --resume`, or ask the user. Never silently take over the work.
   - `status: watch_timeout` (Codex still alive) → this is NOT a failure. The watcher stopped but Codex is still working. Do NOT take over. Do NOT kill the process. Either relaunch `cx-run` to reattach, or inform the user and wait.
8. For resume: use `cx-run --resume <session_id>` (see Resume Protocol). Run with `run_in_background`.
9. For safe mode, add `--sandbox workspace-write --full-auto -c sandbox_workspace_write.network_access=true` instead of `--yolo`. For worktree path, add `-C /absolute/path`.

## Permissions Model

### Default: Full Access (`--yolo`)

Use `--yolo` (alias `--dangerously-bypass-approvals-and-sandbox`) unless the user explicitly requests safe mode. This disables the OS-level sandbox and approval prompts, giving Codex full filesystem/network/process access.

Most local development doesn't need sandboxing, and sandbox restrictions cause failures in multi-agent workflows (blocked sockets, write denials, network timeouts). For untrusted codebases, switch to safe mode.

```bash
codex exec --yolo --skip-git-repo-check \
  -m gpt-5.4 -c model_reasoning_effort='"high"' \
  "$PROMPT" 2>/dev/null
```

### Safe Mode (opt-in)

Use when working with untrusted codebases or when the user explicitly requests sandboxing:

```bash
codex exec --skip-git-repo-check \
  --sandbox workspace-write --full-auto \
  -c sandbox_workspace_write.network_access=true \
  -m gpt-5.4 -c model_reasoning_effort='"high"' \
  "$PROMPT" 2>/dev/null
```

| Scenario | Mode |
| --- | --- |
| Local dev (default) | `--yolo` |
| Untrusted repo/PR | Safe mode |
| User requests sandbox | Safe mode |
| Parallel worktree | `--yolo -C /absolute/path` |
| Resume | `--json --yolo ...` (not inherited) |

## Acceptance Workflow

After Codex execution:

1. **Size-aware diff review:**
   - Start with `git diff --stat` and `--name-only`.
   - **≤10 files**: full diffs per file (`--unified=3`).
   - **>10 files**: stat only + spot-check 3-5 files (`| head -200`).
   - Exclude generated files: `-- ':!package-lock.json' ':!*.lock'`
2. Run tests/checks from Acceptance Criteria.
3. Verify no files outside Ownership/Files boundary were modified.
4. For parallel mode: check `.coord/events.jsonl` (`tail -n 50`) for blockers.
5. Decide: **Accept**, **Revise** (re-enter Execute), or **Abort** (revert).

## Conflict Avoidance

### Serial Mode: Role Locking

- **Explore**: Claude Code is read-only. Codex does not write.
- **Execute**: Codex writes; Claude Code does not change files.
- **Accept**: Claude Code reviews; Codex does not write unless re-entering Execute.

### Parallel Mode

See `references/parallel-mode.md` — File Ownership Locking section.

## Following Up

- If the task package is incomplete, request clarification **before** running Codex.
- After every `codex` command, update `.coord/sessions.jsonl` and confirm next steps with the user.
- Restate the chosen model and reasoning effort when proposing follow-up actions.
- If context is lost, read `.coord/sessions.jsonl` to recover state before taking action.

## Context Protection

- **NEVER** capture raw JSONL into shell variables or read full JSONL files into context. Use `cx-parse extract` or `cx-parse response` for targeted extraction.
- `cx-run` uses `--max-chars 12288` by default — responses longer than this are truncated in the result JSON. The full response remains in the JSONL temp file.
- When reading `.coord/sessions.jsonl`: always use `tail -n 50`, never `cat` the full file.
- When reviewing diffs: use `| head -200` for large outputs.
- For full extraction details, see `references/extraction-patterns.md`.

## Error Handling

- Stop and report on non-zero `codex` exits; request direction before retrying.
- `--yolo` is the default and does not require additional confirmation. Confirm before reverting to `--yolo` from safe mode.
- Summarize warnings/partial results and ask for direction.
- If phase/ownership boundaries are violated, stop and clarify.
- **Parallel mode**: fall back to serial if worktree creation fails.
- **Watch timeout is not a task failure.** `watch_timeout` means the watcher stopped waiting, not that Codex stopped working. The Codex process is still alive. Do NOT treat this as a failure, do NOT kill the process, do NOT take over. Either reattach a watcher or ask the user.
- **Crash recovery**: look up session ID from registry, resume per Resume Protocol. Temporarily remove `2>/dev/null` on first resume after crash.
- **Long-running tasks are normal.** A task taking 15, 30, or 60+ minutes does not justify killing Codex or taking over. The only true failure states are `crashed` (PID dead) and `error`. Everything else means Codex is still working.
