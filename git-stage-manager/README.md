# Git Stage Manager

**Agent-Oriented Version Control Protocol** — a rigorous, state-machine-driven protocol for AI coding agents to make safe, traceable, single-stage Git commits.

## Overview

Git Stage Manager solves a core problem: when an AI agent modifies code, how does it commit exactly its own changes — nothing more, nothing less — while protecting the user's pre-existing uncommitted work?

It defines a **Dual-Track Model**:

- **Agent Intent Layer** (`FILE_SET` + `ACTION_LOG` + `STAGE_CONTEXT`) — the agent declares which files it touched and why
- **Git Source of Truth** — actual repository state validated at every step

## Core Concepts

| Concept | Description |
|---|---|
| **Stage Isolation** | One stage = one commit. No cross-stage leakage. |
| **FILE_SET** | Explicit declaration of new, modified, and deleted files for the stage. |
| **ACTION_LOG** | Record of every file operation with a reason, tied to FILE_SET. |
| **Evidence-Driven Summaries** | Summary files generated from real `git diff`, `cat` output — never from memory. |
| **User Change Protection** | Pre-existing uncommitted changes are never auto-included. |

## State Machine

```
IDLE → PRE_STAGE_CHECK → STAGE_INIT → STAGE_ACTIVE → STAGE_VALIDATE
  → STAGE_SUMMARY → SUMMARY_VALIDATE → STAGE_COMMIT → COMPLETED
                                                      ↓
                                              ERROR / RECOVERY
```

Each state has predefined validation checks and Git commands, ensuring deterministic, auditable execution.

## Allowed Git Commands

Only these commands are permitted:

- `git status`
- `git diff` (against `BASE_COMMIT`)
- `git rev-parse HEAD`
- `git add <file>` (explicit files only)
- `git commit -m "<message>"`
- `git log -1 --format=%H`

Forbidden: `git add .`, `git commit -a`, `git push`, `git reset --hard`, `git stash`.

## Execution Flow

1. **PRE_STAGE_CHECK** — verify no conflicts exist, snapshot existing changes
2. **STAGE_INIT** — record `BASE_COMMIT`, initialize empty `FILE_SET` and `ACTION_LOG`
3. **STAGE_ACTIVE** — make changes, populate `FILE_SET` and `ACTION_LOG`
4. **STAGE_VALIDATE** — assert `FILE_SET` matches actual Git changes exactly
5. **STAGE_SUMMARY** — generate evidence-based summary file
6. **SUMMARY_VALIDATE** — verify summary completeness and correctness
7. **STAGE_COMMIT** — stage declared files + summary, create commit
8. **COMPLETED** — output commit hash and summary path

## Commit Format

```
[<STAGE_ID>][<STAGE_TYPE>] <description>
```

## Usage

This is a **skill** designed for AI coding agents (Claude, Codex, Gemini, OpenCode, etc.). Load the skill when completing a bounded engineering stage that requires a single controlled commit.

## Files

| File | Description |
|---|---|
| `SKILL.md` | Formal protocol specification (English, 398 lines) |
| `summary.md` | Design document v4.0 (Chinese, 493 lines) |

## License

MIT
