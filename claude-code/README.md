Leave a star ⭐ if you like it 😘

# Codex Integration for Claude Code

<img width="2288" height="808" alt="skillcodex" src="https://github.com/user-attachments/assets/85336a9f-4680-479e-b3fe-d6a68cadc051" />


## Purpose
Enable Claude Code to invoke the Codex CLI (`codex exec` and session resumes) for automated code analysis, refactoring, and editing workflows in both serial and parallel worktree collaboration modes.

## Prerequisites
- `codex` CLI installed and available on `PATH`.
- Codex configured with valid credentials and settings.
- Confirm the installation by running `codex --version`; resolve any errors before using the skill.

## Installation

Download this repo and store the skill in ~/.claude/skills/codex

```
git clone --depth 1 git@github.com:skills-directory/skill-codex.git /tmp/skills-temp && \
mkdir -p ~/.claude/skills && \
cp -r /tmp/skills-temp/ ~/.claude/skills/codex && \
rm -rf /tmp/skills-temp
```

## Usage

### Important: Thinking Tokens
By default, this skill suppresses thinking tokens (stderr output) using `2>/dev/null` to avoid bloating Claude Code's context window. If you want to see the thinking tokens for debugging or insight into Codex's reasoning process, explicitly ask Claude to show them.

### Example Workflow

**User prompt:**
```
Use codex to analyze this repository and suggest improvements for my claude code skill.
```

**Claude Code response:**
Claude will activate the Codex skill and:
1. Ask which model to use (default: `gpt-5.3-codex`) unless already specified in your prompt.
2. Ask which reasoning effort level (`xhigh`, `high`, `medium`, or `low`) unless already specified in your prompt.
3. Select appropriate sandbox mode (defaults to `read-only` for analysis)
4. Run a command like:
```bash
codex exec -m gpt-5.3-codex \
  --config model_reasoning_effort="high" \
  --sandbox read-only \
  --full-auto \
  --skip-git-repo-check \
  "Analyze this Claude Code skill repository comprehensively..." 2>/dev/null
```

**Result:**
Claude will summarize the Codex analysis output, highlighting key suggestions and asking if you'd like to continue with follow-up actions.

### Collaboration Modes

This skill supports two collaboration modes:

- `serial` (default): Claude Code and Codex follow a single-lane Explore -> Execute -> Accept workflow.
- `parallel`: Claude Code and Codex run concurrently in separate git worktrees (`cc/<task>` and `cx/<task>`) with explicit file ownership, then merge through `int/<task>`.

Use parallel mode when the task can be cleanly split across independent files/modules and merge overhead is acceptable.

### Session Registry

Codex sessions are tracked in `.coord/sessions.jsonl` (append-only JSONL) so work can be resumed reliably after timeouts, interruptions, or context compression.

Each entry maps a `session_id` to task metadata such as:
- `task`
- `status` (`running`, `paused`, `done`, `failed`)
- `cwd`
- `mode` (`serial` or `parallel`)
- timestamps and optional notes

In parallel mode, each worktree session is recorded separately with its own `cwd` and branch context.

### Resume Protocol

When resuming, the skill uses an explicit session ID from the registry (not `--last`) and sends a self-check prompt first so Codex reports:

1. What is already done
2. Which files changed
3. What remains
4. Any blockers or decisions needed

Resume command pattern:

```bash
echo "<resume prompt>" | codex exec --skip-git-repo-check resume <SESSION_ID> - 2>/dev/null
```

In parallel mode, include `-C <worktree-path>` so the resumed session continues in the correct worktree.

### Detailed Instructions
See `SKILL.md` for complete operational instructions, CLI options, and workflow guidance.
