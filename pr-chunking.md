---
description: Split a large branch diff into smaller, sequential stacked PRs for easier code review
allowed-tools: Bash(git:*), Bash(pnpm:*), Bash(npm:*), Bash(npx:*), Bash(pants:*), Bash(make:*), Bash(cargo:*), Bash(go:*), Bash(pytest:*), Bash(gh:*), Bash(wc:*), Bash(sort:*), Bash(ls:*), Bash(cat:*), Read, Glob, Grep, AskUserQuestion, Skill
---

# PR Chunking

Split a large branch into smaller, sequential stacked PRs that humans can review effectively.

**You MUST run all git commands fresh right now.** Do NOT reuse any previously observed or cached diff, commit log, or branch info from earlier in this conversation.

## Step 0: Parse arguments

Check if the user passed any arguments:

- **`--base <branch>`**: Use `<branch>` as the base for the diff instead of the auto-detected default branch. The next argument after `--base` is the branch name.

**Determine the base branch:**
- If `--base <branch>` was provided, use that value directly.
- Otherwise, auto-detect the default branch by running `git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@'` (if this fails, use `main`).

Use the resulting base branch for all diff commands below. Refer to it as `<base>` in subsequent steps.

## Step 1: Gather git context

Run these commands **in parallel** to gather fresh context:

1. `git branch --show-current` — current branch name (referred to as `<feature-branch>` below)
2. `git diff --stat <base>...HEAD` — diff summary (files + line counts)
3. `git diff --name-status <base>...HEAD` — file change types (A/M/D)
4. `git log --oneline <base>...HEAD` — commits on this branch
5. `git diff --shortstat <base>...HEAD` — total insertions/deletions

From the shortstat output, compute `total_lines = insertions + deletions`.

**Early exit conditions:**
- If there is no diff at all, print: "No changes found on `<feature-branch>` relative to `<base>`. Nothing to split." Then **stop**.
- If `total_lines <= 400`, print: "This branch has ~{total_lines} lines changed — small enough to review as a single PR (threshold: 400 lines). No splitting needed." Then **stop**.

## Step 2: Analyze the diff

### 2a: Classify files

For each changed file from `git diff --name-status`, classify it:

- **Test file**: filename contains `.test.`, `.spec.`, `_test.`, or `_spec.`, OR lives inside a `__tests__/`, `tests/`, or `test/` directory.
- **Config file**: filename matches common config patterns (`package.json`, `*.config.*`, `Makefile`, `BUILD`, `Cargo.toml`, `go.mod`, `pyproject.toml`, `tsconfig.*`, `.eslintrc.*`, `.prettierrc.*`, `Dockerfile`, `docker-compose.*`, `*.toml`, `*.yaml`, `*.yml` in root-level or config directories).
- **Source file**: everything else.

### 2b: Map dependencies between changed files

For each **source file** in the diff, read the file and extract its imports/dependencies. Use language-agnostic patterns — scan for:

- `import ... from '...'` / `import "..."` (JS/TS/Dart)
- `require('...')` / `require "..."` (JS/Ruby)
- `from ... import ...` / `import ...` (Python)
- `use ...` (Rust/PHP)
- `#include "..."` (C/C++)
- `import "..."` / `import (...)` (Go)
- `using ...` (C#)
- Any other import-like statements encountered

Resolve relative paths to determine which **other changed files** each file depends on. Only track dependencies between files that are in the diff — external/third-party dependencies are irrelevant.

### 2c: Build dependency graph

Construct a directed acyclic graph (DAG) where:
- Nodes are changed source files
- Edges point from a file to the files it imports

Compute the **dependency depth** of each file (longest path from any leaf node). Leaves have depth 0.

### 2d: Associate test files

For each test file in the diff, identify its corresponding source file (by naming convention or import analysis). The test file will be grouped with its source file in the same PR.

## Step 3: Design the PR split

### Size targets per PR

These thresholds are based on the SmartBear/Cisco study (2,500 code reviews, 3.2M lines) and Google/Microsoft engineering practices:

| Metric | Target | Soft ceiling | Hard ceiling |
|--------|--------|-------------|-------------|
| Lines (additions + deletions) | 200-400 | 600 (acceptable if mostly new files) | 1,000 (never exceed) |
| Files | ≤10 | 15 | 20 |

**Starting estimate**: `ceil(total_lines / 400)` PRs, then adjust for dependency boundaries.

### Grouping algorithm

1. **PR 1 — Config & tooling**: Group all config file changes. If any source files have zero dependencies AND zero dependents (true leaves), they may also go here if they fit within size targets.

2. **Remaining PRs — by dependency depth**: Starting from depth 0 (leaf source files with no imports from other changed files), group files into PRs:
   - Files at the same dependency depth are candidates for the same PR.
   - If a depth level exceeds the size target, split it into multiple PRs by logical grouping (same directory, same feature area, related functionality).
   - If a depth level is very small, merge it with the next depth level if they fit within size targets.
   - Always include a source file's associated test file(s) in the same PR.

3. **Validation pass**: For each proposed PR, verify:
   - Every import from a changed file is satisfied by either (a) a file already on `<base>`, or (b) a file in the same or a prior PR.
   - No file appears in more than one PR.
   - All test files are grouped with their source files.

4. **Assign slugs**: Give each PR a short (1-3 word) descriptive slug based on its contents (e.g., `types-utils`, `api-layer`, `shared-components`, `history-integration`).

5. **Assign branch names**: `<feature-branch>-pt{N}-{slug}` for each PR. If git rejects this because `<feature-branch>` already exists as a ref (preventing sub-paths), fall back to inserting `-pt{N}-{slug}` before the last path segment.

## Step 4: Auto-detect verification command

Search for a project verification command by checking these indicators **in order** (first match wins). Search from the repository root, and also check in subdirectories where the changed files live:

1. **`package.json`** with a `"checks"` script → `pnpm checks` (or `npm run checks` if no pnpm lockfile)
2. **`package.json`** with a `"validate"` script → `pnpm validate` (or `npm run validate`)
3. **`package.json`** with a `"test"` script → `pnpm test` (or `npm test`)
4. **`BUILD` or `BUILD.pants`** files near changed files → `pants tlc <targets>`
5. **`Makefile`** with a `check`, `verify`, or `test` target → `make check` / `make test`
6. **`Cargo.toml`** → `cargo check && cargo test`
7. **`go.mod`** → `go vet ./... && go test ./...`
8. **`pyproject.toml`** with `[tool.pytest]` or `pytest` dependency → `pytest`

If the changed files span multiple project roots (e.g., a monorepo with multiple `package.json` files), detect the verification command for each project root separately and run them all.

If nothing is detected, set verification to `none` and note this in the plan output.

## Step 5: Present the plan

Output the plan in this format:

```
## PR Chunking Plan

**Branch:** <feature-branch>
**Base:** <base>
**Total:** ~{total_lines} lines across {file_count} files → {N} PRs

### PR Sequence

| # | Branch | Theme | Files | ~Lines | Depends on |
|---|--------|-------|-------|--------|------------|
| 1 | <branch-pt1-slug> | <theme> | <count> | <lines> | — |
| 2 | <branch-pt2-slug> | <theme> | <count> | <lines> | PR 1 |
| ... | ... | ... | ... | ... | ... |

### PR Details

#### PR 1: <theme>
<file list with (new)/(modified)/(deleted) annotations>

#### PR 2: <theme>
<file list>

...

### Verification
Command: <detected command or "none (auto-detection found no project test runner)">

### Size check
- Target: 200-400 lines per PR
- Largest PR: PR {X} at ~{lines} lines {✓ within ceiling / ⚠ exceeds soft ceiling / ✗ exceeds hard ceiling}
```

## Step 6: Prompt — execute?

Use AskUserQuestion to ask: **"Create these {N} stacked branches now?"**

If the user declines, stop. The plan above serves as a guide they can execute manually.

## Step 7: Create stacked branches

Save the current branch name and HEAD SHA so you can return to it after execution.

For each PR (1 through N):

1. **Switch to the correct base:**
   - PR 1: `git checkout <base>`
   - PR N>1: stay on the branch just created (already stacked)

2. **Create the branch:** `git checkout -b <branch-name>`

3. **Checkout files from the feature branch:**
   ```
   git checkout <feature-branch> -- <file1> <file2> ...
   ```
   For deleted files, use `git rm <file>` instead.

4. **Stage and commit:**
   - Stage all relevant files.
   - Write a detailed commit message that:
     - Summarizes what this chunk contains and why it's grouped this way.
     - Ends with `Part {N} of {M} for <ticket-or-feature-description>.`
     - Includes `Co-Authored-By: Claude <noreply@anthropic.com>` on the last line.

5. **Run verification** (if a command was detected):
   - Run the verification command.
   - If it fails, report the error and use AskUserQuestion to ask: **"Verification failed for PR {N}. See error above. Continue anyway, or stop to investigate?"**
   - If the user says stop, check out the original `<feature-branch>` and halt.

6. **Repeat** for the next PR.

After all branches are created, check out the original `<feature-branch>` to restore the user's working state.

Print a summary:
```
Created {N} stacked branches:
  1. <branch-pt1-slug> (from <base>)
  2. <branch-pt2-slug> (from pt1)
  ...
```

## Step 8: Prompt — push?

Use AskUserQuestion to ask: **"Push all {N} branches to remote?"**

If yes: `git push -u origin <branch1> <branch2> ... <branchN>`

If no: print the branch names and stop.

## Step 9: Prompt — create PRs?

Use AskUserQuestion to ask: **"Create GitHub PRs for each branch?"**

If no, print the branch names and stop.

If yes, for each branch (1 through N):

1. **Determine the PR base:**
   - PR 1 targets `<base>` (e.g., `main`)
   - PR N>1 targets the previous PR's branch

2. **Generate the PR description:**
   - Check if the `/pr-description` slash command is available.
   - If available: switch to the branch, then invoke `/pr-description --base <pr-base-branch>` to generate the description. Capture its output.
   - If not available: generate a basic description with a summary section listing the files and a note indicating "Part {N} of {M}".

3. **Create the PR:**
   ```
   gh pr create --base <pr-base-branch> --head <branch-name> --title "<title>" --body "<description>"
   ```
   - Title format: `<theme> (part {N}/{M} of <ticket-or-feature>)`
   - Include "Part {N} of {M}" in the body along with the generated description.

4. **Collect the PR URL** from the output.

After all PRs are created, print a summary:
```
Created {N} PRs:
  1. <PR-URL-1> — <theme>
  2. <PR-URL-2> — <theme>
  ...

Merge order: 1 → 2 → 3 → ... → {N}
After merging each PR, the next PR's diff will shrink to just its own changes.
```
