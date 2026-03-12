# Parallel Worktree Mode

Claude Code and Codex work simultaneously in separate git worktrees on different branches, then merge via an integration branch.

## When to Use

- Task is naturally splittable into 2+ independent modules/files.
- No tight coupling between the subtasks (minimal shared files).
- Time savings from parallelism outweigh the merge overhead.

## Workflow

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

## Bootstrap Commands

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
  -m gpt-5.4 -c model_reasoning_effort='"high"' \
  --yolo \
  "$PROMPT" >"$JSONL" 2>/dev/null
SESSION_ID="$($CX session-id "$JSONL")"
RESPONSE="$($CX response "$JSONL" --max-chars 12288)"
```

## Integration Commands

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

## File Ownership Rules

- Each file/directory is assigned to **exactly one agent** during the Plan phase.
- Shared files (e.g., routes, schemas, lockfiles, config) must be flagged as **"serial handoff"** — only one agent may edit them at a time, with an explicit handoff.
- If an agent discovers it needs to modify a file owned by the other, it must **stop and signal** rather than edit directly.

## Worktree Session Management

Uses the shared Session Registry (`.coord/sessions.jsonl`). In parallel mode, each worktree's Codex session gets its own registry entry with `mode: parallel` and the worktree `cwd`.

Resume Codex in its worktree using the recorded session ID. Note: `resume` does NOT support `-C` — you must `cd` first:

```bash
CX="$HOME/.claude/skills/codex/scripts/cx-parse"
JSONL="$(mktemp -t codex.XXXXXX.jsonl)"
WORKTREE="$(cd ../<project>-codex && pwd)"
cd "$WORKTREE" && codex exec --json --yolo \
  --skip-git-repo-check resume <SESSION_ID> "<resume prompt>" >"$JSONL" 2>/dev/null
SESSION_ID="$($CX session-id "$JSONL")"
RESPONSE="$($CX response "$JSONL" --max-chars 12288)"
```

## Coordination Protocol (`.coord/`)

Use a `.coord/` directory in the project root to coordinate state between agents. This directory is gitignored via `.git/info/exclude` during bootstrap (see Bootstrap Commands).

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

Check `.coord/events.jsonl` after Codex completes to understand status and blockers.

## Integration Gate

Before merging into `int/<task>`:
1. Run the overlap check from Integration Commands. If any files appear in both diffs, stop and resolve ownership before merging.
2. If overlap is empty, proceed with the merge.
3. Run the full test suite on the integration branch.
4. Verify all acceptance criteria from both subtask packages.

## File Ownership Locking (Conflict Avoidance)

- Each file/module is assigned to one agent in the Task Package `Ownership` field.
- Both agents may write simultaneously, but **only to their owned files**.
- Shared files require explicit **claim/release** via `.coord/claims.yaml`.
- Before editing a shared file, check that no other agent holds a claim on it.
- If ownership is unclear, stop and request clarification.

## Cleanup

After successful integration:
```bash
git worktree remove ../<project>-codex
git branch -d cc/<task> cx/<task> int/<task>
rm -rf .coord/
```
