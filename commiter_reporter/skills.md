# Git Stage Manager Skill

## Purpose

Use this skill to manage a single stage of an engineering project with Git-based isolation, traceability, and safe commit behavior.

This skill helps an AI coding agent complete one bounded engineering stage, generate a stage summary, and create exactly one Git commit without accidentally including unrelated user changes.

## When to Use

Use this skill when the user asks the agent to do any of the following:

- Complete a clearly bounded engineering stage
- Implement a feature as a staged commit
- Refactor part of a project and commit the result
- Make a structural or architectural change that should be traceable
- Produce a summary file for the completed stage
- Safely commit only the files modified during the current stage

## Do Not Use

Do not use this skill when:

- The user asks for casual advice, explanation, or code review only
- The task is a small one-off edit that does not require a commit
- The user explicitly says not to commit
- The repository is not under Git control
- The user asks to push changes remotely
- The task requires handling multiple independent stages at once

## Core Principle

Git is the source of truth. The agent records intent.

```text
Intent Layer  = FILE_SET + ACTION_LOG
Source Truth  = Git repository state
```

The agent must never rely only on memory. Git status and diff must be used at key checkpoints to validate the repository state.

## Stage Context

At the beginning of every stage, initialize the following context:

```text
STAGE_ID      = UUID or short unique identifier
BASE_COMMIT   = output of `git rev-parse HEAD`
STAGE_TYPE    = feature | fix | refactor | chore
FILE_SET      = files explicitly owned by this stage
ACTION_LOG    = operation records for this stage
```

`FILE_SET` is divided into:

```text
STEP_NEW_FILES
STEP_MODIFIED_FILES
STEP_DELETED_FILES
```

## File Set Rules

### Weak Lock Rule

`FILE_SET` is locked by default but can be explicitly expanded.

Allowed:

```text
ADD_TO_FILE_SET(file, action, reason)
```

A file may be added to `FILE_SET` only when:

1. The current stage directly creates, modifies, or deletes it; or
2. The user explicitly asks the agent to include it.

Disallowed:

- Implicitly adding files discovered from `git status`
- Automatically adding all changed files
- Adding unrelated user changes
- Using semantic guessing alone to decide that an existing change is related

## Action Log

For every file operation, record an action log entry:

```text
{
  file: "path/to/file",
  action: "create | modify | delete",
  reason: "why this file belongs to the current stage"
}
```

The action log is used to generate the summary and validate the commit scope.

## Git Command Rules

Allowed:

```bash
git status
git diff
git diff BASE_COMMIT -- <file>
git rev-parse HEAD
git add <explicit-file>
git commit -m "<message>"
```

Forbidden unless the user explicitly requests otherwise:

```bash
git add .
git add -A
git commit -a
git push
git reset --hard
git clean -fd
```

## State Machine

```text
IDLE
  -> PRE_STAGE_CHECK
  -> STAGE_INIT
  -> STAGE_ACTIVE
  -> STAGE_VALIDATE
  -> STAGE_SUMMARY
  -> STAGE_COMMIT
  -> COMPLETED

Any state
  -> ERROR
  -> RECOVERY
```

## Execution Procedure

### 1. PRE_STAGE_CHECK

Run:

```bash
git status
```

Behavior:

- Detect existing uncommitted changes.
- Do not automatically include existing changes.
- Existing changes remain outside `FILE_SET` unless the current stage directly modifies them or the user explicitly asks to include them.
- If existing changes conflict with the stage, ask the user before proceeding.

### 2. STAGE_INIT

Run:

```bash
git rev-parse HEAD
```

Initialize:

```text
STAGE_ID
BASE_COMMIT
STAGE_TYPE
FILE_SET = empty
ACTION_LOG = []
```

### 3. STAGE_ACTIVE

Perform the requested engineering work.

For each file operation:

1. Add the file to the correct `FILE_SET` bucket.
2. Add an `ACTION_LOG` entry.
3. Keep the stage focused on the requested scope.

Example:

```text
ADD_TO_FILE_SET("src/parser.ts", "modify", "update parser to support the new stage summary format")
```

### 4. STAGE_VALIDATE

Run:

```bash
git status
git diff
```

Validate all of the following:

```text
1. Every source file changed by this stage is in FILE_SET.
2. Every source file in FILE_SET has a matching reason in ACTION_LOG.
3. No unrelated source file is staged or prepared for commit.
4. Untracked source files are either in FILE_SET or explicitly ignored.
5. Generated/cache/build artifacts are ignored unless explicitly part of the stage.
```

If validation fails, enter `ERROR`.

### 5. STAGE_SUMMARY

Create a summary file named:

```text
step_<stage_id>_<short_description>.md
```

The summary must contain:

```text
1. Stage description
2. STAGE_ID
3. STAGE_TYPE
4. BASE_COMMIT
5. FILE_SET
6. ACTION_LOG summary
7. Key diff summary
8. Risks or notes
```

The summary should be concise. Do not paste full file contents unless the user explicitly asks.

For changed files, inspect diffs with:

```bash
git diff BASE_COMMIT -- <file>
```

### 6. STAGE_COMMIT

Stage only explicit files:

```bash
git add <FILE_SET files>
git add <summary file>
```

Then commit:

```bash
git commit -m "[<STAGE_ID>][<STAGE_TYPE>] <short description>"
```

Never run:

```bash
git add .
git add -A
git commit -a
```

### 7. COMPLETED

After a successful commit, report:

```text
1. Summary file path
2. Commit hash
3. Brief stage summary
```

Do not push.

## Error Handling

Enter `ERROR` when:

- `FILE_SET` does not match Git changes
- Git command fails
- There are untracked source files not accounted for
- There are unrelated staged files
- The working tree contains conflicting user changes

Recovery options:

```text
1. Update FILE_SET explicitly, with reasons
2. Ask the user how to handle conflicting changes
3. Abort the stage without committing
4. Leave the repository unchanged when safe recovery is not possible
```

Do not use destructive recovery commands such as `git reset --hard` or `git clean -fd` unless the user explicitly instructs it.

## Output Format

When the stage completes, respond with:

```text
Completed stage: <short description>
Summary file: <path>
Commit: <hash>
Notes: <short notes, if any>
```

If the stage cannot be completed, respond with:

```text
Stage not committed.
Reason: <specific reason>
Required action: <what needs to happen next>
```

## Minimal Example

User request:

```text
Implement the new parser config format as one stage and commit it.
```

Expected behavior:

```text
1. Run git status.
2. Initialize STAGE_ID and BASE_COMMIT.
3. Modify only required parser/config files.
4. Record each file in FILE_SET and ACTION_LOG.
5. Validate with git status and git diff.
6. Generate step_<stage_id>_parser_config.md.
7. git add only FILE_SET files and the summary file.
8. git commit once.
9. Report summary path and commit hash.
```

## Agent Reminder

```text
Commit is the stage boundary.
FILE_SET is explicit intent.
Git is the source of truth.
Never commit unrelated user changes.
Never use git add .
Never push unless explicitly requested.
```
