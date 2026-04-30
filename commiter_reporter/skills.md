# 📄 `skills.md`（工程阶段管理）

````markdown
---
name: engineering_stage_commit
description: |
  Manage stage-based commits for engineering projects.
  Ensure strict stage isolation, controlled file scope, and structured step reports.
---

# Engineering Stage Commit Skill

## 🎯 Purpose

Enforce structured stage-based workflow:

- Each stage = one isolated commit
- Generate a step report file
- Avoid cross-stage pollution
- Never commit unrelated changes
- Never push unless explicitly required

---

# 🧠 Core Rules (STRICT)

## 1. Commit Rules

Allowed:
- git commit

Forbidden:
- git push (unless user explicitly requests)
- git add .
- git add -A
- git commit -a

---

## 2. File Scope Control (CRITICAL)

You MUST explicitly maintain stage file sets during execution:

```text
STEP_NEW_FILES
STEP_MODIFIED_FILES
STEP_DELETED_FILES
````

Rules:

* Only files you created/modified in THIS stage can be included
* DO NOT infer file list from git status or full diff
* DO NOT scan entire workspace to decide scope

---

## 3. User Changes Protection (CRITICAL)

At stage start, there may be existing uncommitted changes:

| Case                     | Action                 |
| ------------------------ | ---------------------- |
| Related to current stage | Include in this stage  |
| Unrelated                | Ignore (DO NOT commit) |
| Uncertain                | Ignore by default      |

NEVER commit user changes blindly.

---

## 4. Diff Isolation (CRITICAL)

All diffs must represent ONLY this stage.

Use:

```bash
git diff BASE_COMMIT -- <file>
```

DO NOT:

* use full `git diff`
* include previous stage changes

---

## 5. Stage Isolation

* Each stage MUST end with one commit
* One commit = one stage
* NEVER mix multiple stages in one commit

---

## 6. Non-stage Work

* Normal small edits DO NOT trigger commit
* They may be:

  * absorbed into next stage (if relevant)
  * ignored (if unrelated)

---

# 🔄 State Machine

## States

```text
IDLE
PRE_STAGE_CHECK
STAGE_ACTIVE
STAGE_SUMMARY
STAGE_COMMIT
COMPLETED
```

---

## Transitions

```text
IDLE → PRE_STAGE_CHECK → STAGE_ACTIVE → STAGE_SUMMARY → STAGE_COMMIT → COMPLETED
```

---

# ⚙️ Execution Steps

## Step 0: Start Stage

Record base commit:

```bash
BASE_COMMIT=$(git rev-parse HEAD)
```

Initialize:

```text
STEP_NEW_FILES = []
STEP_MODIFIED_FILES = []
STEP_DELETED_FILES = []
```

---

## Step 1: PRE_STAGE_CHECK

Check existing changes:

* Decide per file:

  * include (if relevant)
  * ignore (if unrelated)

DO NOT commit them automatically.

---

## Step 2: STAGE_ACTIVE

During work:

* When creating files → add to STEP_NEW_FILES
* When modifying files → add to STEP_MODIFIED_FILES
* When deleting files → add to STEP_DELETED_FILES

This tracking is MANDATORY.

---

## Step 3: STAGE_SUMMARY

Generate report file:

### Filename

```text
stepX_Y_<description>.txt
```

---

## Report Content

### 1. Stage Description

Explain:

* purpose
* main changes
* relation to previous work

---

### 2. New Files

```bash
echo "## New Files" >> FILE
```

---

### 3. Full Content of New Files

```bash
for f in ${STEP_NEW_FILES[@]}; do
  cat "$f" >> FILE
done
```

DO NOT manually write code.

---

### 4. Modified Files

```bash
echo "## Modified Files" >> FILE
```

---

### 5. Diff of Modified Files

```bash
for f in ${STEP_MODIFIED_FILES[@]}; do
  git diff "$BASE_COMMIT" -- "$f" >> FILE
done
```

---

## Step 4: STAGE_COMMIT

Only add stage files:

```bash
git add <STEP_NEW_FILES>
git add <STEP_MODIFIED_FILES>
git add <STEP_DELETED_FILES>
git add <REPORT_FILE>
```

Commit:

```bash
git commit -m "stepX_Y_<description>"
```

---

## Step 5: Output

Return:

* Report file path
* Short summary of the stage

---

# 🚫 Forbidden Actions

* git add .
* git add -A
* git commit -a
* git push
* committing unrelated files
* generating diff from full workspace
* skipping stage commit
* mixing multiple stages

---

# ✅ Self Checklist (MUST PASS)

Before commit, verify:

* [ ] File list is explicitly tracked (NOT inferred)
* [ ] No unrelated files included
* [ ] Diff only includes stage changes
* [ ] New file content is from actual files (not handwritten)
* [ ] No use of git add .
* [ ] Exactly one commit for this stage
* [ ] Report file generated
* [ ] No git push executed

````