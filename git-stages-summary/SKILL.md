---
name: git-stages-summary
description: Consolidate multiple stages into a lightweight release summary for web LLMs. Per-stage sections show only description + file list + ACTION_LOG; the complete git diff from start to end serves as the single authoritative evidence. Use when multiple stages have been completed and a token-efficient holistic view is needed.
---

# Git Stages Summary Skill

## Overview

This skill aggregates **multiple bounded engineering stages** into a single, lightweight release summary optimized for web LLM consumption.

After several stages have been completed, loading each individual `step_*.md` into a web LLM becomes expensive in tokens and cumbersome to manage. This skill solves that by producing a **two-layer document**:

- **Layer 1 (lightweight per-stage)**: each stage summarized in ~3–5 lines — description, file list, ACTION_LOG only
- **Layer 2 (single authoritative evidence)**: the complete `git diff <START>..<END>` — one unified diff covering the entire release range

The web LLM reads Layer 1 to understand the development journey, then reads Layer 2 once to grasp the full code delta. No need to open N separate `step_*.md` files.

It operates at a higher level than `git-stage-manager`: one release summary spans N stages.

---

## Core Model

### Dual-Track Model (Release Level)

```text
Release Evidence Layer = AGGREGATED_DIFF + ACTION_LOG_SUMMARY + FILE_CHANGE_CATALOG
Git Source of Truth   = actual repository state across the range
```

### Release Context

```text
RELEASE_ID
RELEASE_START_COMMIT
RELEASE_END_COMMIT
RANGE_STAGES        # list of STAGE_IDs covered
AGGREGATED_DIFF     # git diff from start to end
LOG_AGGREGATION     # ACTION_LOG entries from all stages
FILE_CHANGE_CATALOG # categorized files across all stages
RELEASE_SUMMARY_FILE
FILTER_CONFIG       # user-defined filter rules (see below)
```

---

### FILTER_CONFIG

User-defined filtering rules that control which files appear in the release summary. Defined via prompt at invocation time.

```text
FILTER_CONFIG = {
  exclude_patterns: ["*.md", "*.json", ".gitignore", ...],
  exclude_reasons: ["config", "documentation", "dependency", ...],
  include_only: null  # if set, only these patterns are included
}
```

Evaluation rules:

```text
1. If include_only is set, a file must match at least one include pattern to appear
2. If a file matches any exclude_pattern, it is omitted from all sections
3. exclude_reasons filters by ACTION_LOG reason text (e.g., skip all "config" entries)
4. Patterns are gitignore-style glob patterns
5. FILTER_CONFIG applies only to the release summary — it does not alter Git state or stage summaries
```

Usage examples:

```text
# Exclude generated/doc files
FILTER_CONFIG: exclude_patterns = ["*.md", "*.json", "*.snap", "*.map", ".gitignore"]

# Focus only on source code
FILTER_CONFIG: exclude_patterns = ["*.md", "*.json", "*.yaml", "*.config.*"]

# Only show business logic changes
FILTER_CONFIG: exclude_patterns = ["*.md", "*.json", "*.test.*", "*.spec.*", ".gitignore"]

# Include only specific areas
FILTER_CONFIG: include_only = ["src/*", "lib/*"]
```

---

## State Machine

```text
IDLE
  -> RELEASE_INIT
  -> RELEASE_COLLECT
  -> RELEASE_AGGREGATE_DIFF
  -> RELEASE_AGGREGATE_LOG
  -> RELEASE_SYNTHESIS
  -> RELEASE_VALIDATE
  -> COMPLETED

Any state
  -> ERROR
  -> RECOVERY
```

---

## Execution Procedure

### 1. RELEASE_INIT

```bash
RELEASE_START_COMMIT=$(git rev-parse <start_ref>)
RELEASE_END_COMMIT=$(git rev-parse HEAD)
```

If called after stages have just finished, `RELEASE_START_COMMIT` should be the `BASE_COMMIT` of the first stage in the range (or an explicit tag/branch reference).

Initialize:

```text
RELEASE_ID
RELEASE_START_COMMIT
RELEASE_END_COMMIT
RANGE_STAGES = []
AGGREGATED_DIFF = unset
LOG_AGGREGATION = []
FILE_CHANGE_CATALOG = { new: [], modified: [], deleted: [] }
RELEASE_SUMMARY_FILE = unset
FILTER_CONFIG = user-defined patterns from prompt
```

---

### 2. RELEASE_COLLECT

Collect existing stage summary files within the commit range:

```bash
git log --oneline $RELEASE_START_COMMIT..$RELEASE_END_COMMIT
git log --format="%H" $RELEASE_START_COMMIT..$RELEASE_END_COMMIT
```

Rules:

```text
1. Find all step_*.md files referenced in commits within range
2. Read each summary file
3. Extract STAGE_ID, STAGE_TYPE, FILE_SET, ACTION_LOG from each
4. Populate RANGE_STAGES
```

---

### 3. RELEASE_AGGREGATE_DIFF

```bash
git diff $RELEASE_START_COMMIT..$RELEASE_END_COMMIT
git diff $RELEASE_START_COMMIT..$RELEASE_END_COMMIT --stat
```

Rules:

```text
1. Capture diff across the entire range (full and --stat)
2. Store as AGGREGATED_DIFF (evidence, not paraphrase)
3. Apply FILTER_CONFIG to exclude matched files from the catalog:
   - Remove files matching exclude_patterns from FILE_CHANGE_CATALOG
   - Remove files matching include_only (if set, drop non-matching files)
   - Remove entries matching exclude_reasons from LOG_AGGREGATION
```

Filtering does NOT change the `git diff` output itself — it controls which files appear in the synthesized summary sections. Actual diff content for included files is still fetched per-file on demand in RELEASE_SYNTHESIS.

---

### 4. RELEASE_AGGREGATE_LOG

From each collected stage summary, extract its `ACTION_LOG` entries and merge:

```text
LOG_AGGREGATION = [
  {
    stage: "stage_01",
    files: [...],
    intent: "description from stage summary"
  },
  ...
]
```

Rules:

```text
1. Preserve per-stage grouping (do not flatten)
2. Preserve original ACTION_LOG reasons verbatim
3. If a file appears in multiple stages, list all touch entries
```

---

### 5. RELEASE_SYNTHESIS

Create:

```text
release_<release_id>_<description>.md
```

Content structure:

```text
# Release Summary

## 1. Release Metadata
- RELEASE_ID:
- RELEASE_START_COMMIT:
- RELEASE_END_COMMIT:
- STAGES_COVERED:
- TIMESTAMP:
- STATS (filtered): files changed, insertions, deletions

## 2. High-Level Intent

Aggregate of ACTION_LOG intents across all stages.
Provide a paragraph summary of what this release accomplishes as a whole.

## 3. Stage-by-Stage Overview (LIGHTWEIGHT)

For each stage in RANGE_STAGES:

  ### Stage <STAGE_ID> [<STAGE_TYPE>]
  - **Description**: one-liner of what this stage achieved
  - **Files**: new / modified / deleted (file names only, no diffs)
  - **ACTION_LOG**:
    ```
    file | action | reason
    ```

No per-stage code content or diffs here. Keep each stage to ~3-5 lines.
The goal is to give the web LLM a quick map of the development journey.

## 4. Complete Diff (EVIDENCE)

```
git diff $RELEASE_START_COMMIT..$RELEASE_END_COMMIT
```

Full unified diff from release start to end. This is the single authoritative code change — the web LLM reads this once to understand the entire codebase delta.

## 5. Risks / Notes
```

Rules:

```text
- Section 3 (per-stage) must be lightweight: file names + ACTION_LOG only
- Section 4 (diff) must be the COMPLETE unified git diff — not per-file, not summarized
- Section 4 is the single authoritative evidence section
- FILTER_CONFIG filters files from Section 3 file lists and Section 4 diff output
- All evidence must be generated via tool/command, not reconstructed from memory
```

---

### Evidence Generation Rule (CRITICAL)

Same as `git-stage-manager`:

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
```

---

### 6. RELEASE_VALIDATE

```text
1. All RANGE_STAGES have their ACTION_LOG extracted
2. AGGREGATED_DIFF matches the sum of all stage summaries
3. Every file in aggregated diff is accounted for in at least one stage
4. RELEASE_SUMMARY_FILE not part of any stage
5. Section 3 (per-stage) contains only lightweight info — no code content or per-file diffs
6. Section 4 (complete diff) is a single unified git diff from start to end
7. Section 4 diff output is verified against git (matches actual diff)
8. FILTER_CONFIG was applied consistently across all sections
9. No files matching exclude_patterns appear in the output
10. If include_only set, all output files match at least one include pattern
```

If failed:

```text
→ fix summary before completion
```

---

### 7. COMPLETED

Output:

```text
Completed release summary: <description>
Summary file: <path>
Range: <start_commit>..<end_commit>
Stages: <count>
Files changed: <stat>
```

Do not push.

---

## Error Handling

Enter ERROR if:

```text
- Commit range is empty or invalid
- Summary files not found or unreadable
- AGGREGATED_DIFF does not match stage FILE_SETs
- Release summary validation fails
```

---

## Recovery

```text
1. Verify commit range
2. Re-read unavailable summary files
3. Fix release summary explicitly
4. Re-run validation
5. Ask user if needed
```

Do not use destructive commands.

---

## Git Command Policy

### Allowed

```bash
git rev-parse
git log
git diff
git diff --stat
git diff <start>..<end> -- <file>
cat <file>
```

### Forbidden

```bash
git add .
git commit
git push
git reset --hard
git clean -fd
git stash
```

Reason:

```text
This skill is read-only with respect to repository state. It produces a summary document but does not create commits or modify the working tree (unless explicitly instructed by the user).
```

---

## Anti-Patterns

```text
- Skipping stage summary collection
- Reconstructing ACTION_LOG from memory instead of reading stage summaries
- Including per-file diffs in the per-stage sections (Section 3 must be lightweight)
- Splitting the complete diff into per-file diffs (Section 4 must be unified)
- Writing subjective conclusions not supported by evidence
- Treating release summary as a replacement for individual stage commits
- Applying FILTER_CONFIG inconsistently across sections
- Silently dropping files without noting the filter reason
```

---

## Agent Reminder

```text
release = stage range boundary
per-stage = lightweight map (names + actions only)
AGGREGATED_DIFF = single authoritative evidence
Git = truth
FILTER_CONFIG = user focus scope

Collect all stage summaries before synthesis
Never bloat per-stage sections with code or diffs
Section 4 must be COMPLETE unified git diff
Never reconstruct evidence from memory
Apply filters consistently across all sections
Validate before completion
```
