---
name: git-stage-manager
description: Orchestrate stage-based Git commits for engineering projects with strict isolation, explicit file ownership, and safe commit behavior. Use when completing a bounded engineering stage that requires a single controlled commit, structured summary generation, and protection against unintended changes.
---

# Git Stage Manager Skill

## Overview

This skill manages **one bounded engineering stage** in a Git repository.

It implements an **Agent-Oriented Version Control Protocol**: the agent may modify code, generate a stage summary, and create exactly one Git commit while avoiding accidental inclusion of unrelated user changes.

This is a Phase 1 single-skill implementation.

---

## Core Model

### Dual-Track Model

```text
Agent Intent Layer = FILE_SET + ACTION_LOG + STAGE_CONTEXT
Git Source of Truth = actual repository state
````

---

## Stage Context

```text
STAGE_ID
BASE_COMMIT
STAGE_TYPE
FILE_SET
ACTION_LOG
SUMMARY_FILE
```

---

## FILE_SET Structure

```text
STEP_NEW_FILES
STEP_MODIFIED_FILES
STEP_DELETED_FILES
```

---

## State Machine

```text
IDLE
  -> PRE_STAGE_CHECK
  -> STAGE_INIT
  -> STAGE_ACTIVE
  -> STAGE_VALIDATE
  -> STAGE_SUMMARY
  -> SUMMARY_VALIDATE
  -> STAGE_COMMIT
  -> COMPLETED

Any state
  -> ERROR
  -> RECOVERY
```

---

## Execution Procedure

### 1. PRE_STAGE_CHECK

```bash
git status
```

Rules:

```text
- Do not include existing changes automatically
- Do not stage anything
- Leave unrelated user changes untouched
- If conflicts exist, ask user or abort safely
```

---

### 2. STAGE_INIT

```bash
BASE_COMMIT=$(git rev-parse HEAD)
```

Initialize:

```text
STAGE_ID
BASE_COMMIT
STAGE_TYPE
FILE_SET = empty
ACTION_LOG = []
SUMMARY_FILE = unset
```

---

### 3. STAGE_ACTIVE

For every file operation:

```text
1. Add file to FILE_SET
2. Record ACTION_LOG
```

Example:

```text
ADD_TO_FILE_SET(
  file="src/parser.ts",
  action="modify",
  reason="update parser logic"
)
```

Rules:

```text
- No implicit file inclusion
- No scope expansion without reason
```

---

### 4. STAGE_VALIDATE

```bash
git status
git diff
```

Validation:

```text
1. FILE_SET ⊆ git changes
2. git changes ⊆ FILE_SET (for source files)
3. Every FILE_SET entry has ACTION_LOG
4. No untracked source files outside FILE_SET
5. No unrelated staged files
```

If failed:

```text
→ ERROR
```

---

### 5. STAGE_SUMMARY

Create:

```text
step_<stage_id>_<description>.md
```

Content:

```text
# Stage Summary

## 1. Stage Description

## 2. Stage Metadata
- STAGE_ID:
- STAGE_TYPE:
- BASE_COMMIT:

## 3. New Files

## 4. New File Full Contents
Use real file content via tool/command/script

## 5. Modified Files

## 6. Modified File Diffs
Use git diff against BASE_COMMIT

## 7. Deleted Files Optional

## 8. ACTION_LOG

## 9. Risks / Notes
```

Rules:

```text
- Must include full content for new files
- Must include BASE_COMMIT diffs for modified files
- Do not reconstruct content manually
- Do not include full-repo diff
```

---

### Special Rule: Summary File Exclusion

```text
SUMMARY_FILE is not a stage-owned file.

- Do not include it in FILE_SET
- Do not include it under New File Full Contents
- Prevent recursive expansion
```

---

### Evidence Generation Rule (CRITICAL)

For New File Full Contents and Modified File Diffs:

```text
MUST:
- Be generated via tool, command, or script output
- Reflect real filesystem or Git state

Acceptable methods:
- shell commands (cat, git diff, etc.)
- redirection (>>)
- custom scripts
- trusted file read tools

MUST NOT:
- manually write code content
- reconstruct content from memory
- paraphrase or summarize file contents

The output must reflect actual state at generation time.
```

---

### 6. SUMMARY_VALIDATE

```text
1. All new files listed
2. All new files include full content (via evidence rule)
3. All modified files listed
4. All modified files include BASE_COMMIT diffs
5. Deleted files handled if exist
6. No unrelated files included
7. No full-repo diff
8. SUMMARY_FILE not recursively embedded
```

If failed:

```text
→ fix summary before commit
```

---

### 7. STAGE_COMMIT

Before commit, perform lightweight final check:

```bash
git status
```

Then:

```bash
git add <FILE_SET files>
git add <SUMMARY_FILE>
git commit -m "[<STAGE_ID>][<STAGE_TYPE>] <description>"
```

Notes:

```text
- Do not re-run full diff validation here (already done in STAGE_VALIDATE)
- Keep final check minimal to reduce overhead
```

---

### 8. COMPLETED

Output:

```text
Completed stage: <description>
Summary file: <path>
Commit: <hash>
Notes: <optional>
```

Do not push.

---

## Error Handling

Enter ERROR if:

```text
- FILE_SET mismatch
- git failure
- untracked source files
- unrelated staged files
- summary validation failure
```

---

## Recovery

```text
1. Fix FILE_SET explicitly
2. Re-run validation
3. Ask user if needed
4. Abort commit safely if unresolved
```

Do not use destructive commands.

---

## Git Command Policy

### Allowed

```bash
git status
git diff
git diff BASE_COMMIT -- <file>
git rev-parse HEAD
git add <file>
git commit -m ""
git log -1 --format=%H
```

### Forbidden

```bash
git add .
git add -A
git commit -a
git push
git reset --hard
git clean -fd
git stash
```

Reason:

```text
These commands may introduce unrelated changes, destroy state, or break traceability.
```

---

## Anti-Patterns

```text
- Using git add .
- Including files from git status automatically
- Committing unrelated changes
- Skipping summary
- Writing fake or reconstructed file content
- Using full-repo diff
- Recursive summary inclusion
```

---

## Agent Reminder

```text
commit = stage boundary
FILE_SET = intent
ACTION_LOG = reasoning
Git = truth

Never commit unrelated user changes
Never use git add .
Never push unless explicitly requested
Validate before commit
```

```
