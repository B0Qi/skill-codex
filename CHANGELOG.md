# Changelog

All notable changes to the Codex skill for Claude Code.

## [Unreleased]

### Added
- **Parallel Worktree Mode**: Claude Code and Codex can now work simultaneously in separate git worktrees on different branches, with file ownership locking and an integration gate for safe merging.
- **Session Registry** (`.coord/sessions.jsonl`): Persistent mapping of Codex session IDs to task descriptions, enabling recovery from timeouts, context loss, and conversation compression.
- **Resume Protocol**: Standardized procedure for resuming Codex sessions — requires self-check summary (completed work, modified files, remaining tasks, blockers) before continuing.
- **Coordination Protocol** (`.coord/`): Directory structure for parallel mode coordination including `plan.yaml`, `claims.yaml`, `sessions.jsonl`, and `events.jsonl`.
- **Integration Gate**: Pre-merge overlap check between parallel branches to prevent silent conflicts.
- **Bootstrap Commands**: Automated setup for worktrees, branches, and `.coord/` directory with proper gitignore via `.git/info/exclude`.
- **File Ownership Rules**: Each file assigned to exactly one agent; shared files require explicit serial handoff.

### Changed
- **Collaboration Modes**: Restructured from single "Collaboration Paradigm" to dual-mode selection (Serial Mode + Parallel Worktree Mode).
- **Task Package Format**: Added `Mode`, `Branch`, `Worktree`, and `Ownership` fields for parallel mode support.
- **Default resume strategy**: Changed from `resume --last` to `resume <SESSION_ID>` for reliability across worktrees.
- **Sandbox selection**: Removed `--full-auto` from `danger-full-access` to avoid semantic conflicts.
- **Error handling**: Added stderr capture on non-zero exit, timeout recovery via session registry, and parallel mode fallback.
- **Acceptance workflow**: Now checks file ownership boundaries and `.coord/events.jsonl` for parallel mode blockers.

### Fixed
- `resume` stdin syntax: added required `-` parameter for stdin prompt piping.
- Terminology consistency: unified "forbidden files" to "Ownership / Files boundary".

## [1.0.0] - 2026-02-06

### Added
- Three-phase collaboration paradigm: Explore (Claude Code) → Execute (Codex) → Accept (Claude Code).
- Standardized Task Package format with Goal, Files, Constraints, Acceptance Criteria, and Notes.
- Smart Sandbox Selection with least-privilege defaults.
- Role Locking for conflict avoidance across phases.
- Session resume support via `codex exec resume --last`.
- Thinking token suppression (`2>/dev/null`) by default.
- Model selection between `gpt-5.3-codex` and `gpt-5.3` with reasoning effort levels.

## [0.x] - Pre-collaboration

- Basic Codex CLI integration (`codex exec`).
- Manual model and sandbox configuration.
- README and installation instructions.
