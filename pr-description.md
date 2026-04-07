---
description: Generate a PR description from the current branch diff, using the repo's PR template if available
allowed-tools: Bash(git:*), Read, Glob
---

# Generate PR Description

Generate a concise, template-aware PR description from the current branch's diff. Print it as raw markdown to the terminal for clean copy-paste into GitHub.

**You MUST run all git commands fresh right now.** Do NOT reuse any previously observed or cached diff, commit log, or branch info from earlier in this conversation.

## Step 1: Parse arguments

Check if the user passed any arguments:

- **Ticket ID**: Any argument matching `[A-Z]+-\d+` (e.g., `MX2-1234`, `PROJ-567`). Store it for later.
- **`--base <branch>`**: Use `<branch>` as the base for the diff instead of the auto-detected default branch. The next argument after `--base` is the branch name.
- **`--verbose`**: Produce more detailed change descriptions, mention specific files by name.
- **`--brief`**: Produce shorter output — summary paragraph + flat bullet list only, no section headers.
- If both `--verbose` and `--brief` are passed, `--brief` wins.

## Step 2: Gather git context

Run these commands **in parallel** to gather fresh context:

1. `git rev-parse --show-toplevel` — repo root path
2. `git branch --show-current` — current branch name
3. **Determine the base branch:**
   - If `--base <branch>` was provided in Step 1, use that value directly and skip auto-detection.
   - Otherwise, run `git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@'` to auto-detect the default branch (if this fails, use `main`).
4. `git log <base-branch>...HEAD --oneline` — commits on this branch
5. `git diff <base-branch>...HEAD` — committed changes vs base branch
6. `git diff` — unstaged changes (working directory)
7. `git diff --cached` — staged but uncommitted changes

Where `<base-branch>` is either the `--base` value or the auto-detected default branch.

The **full diff** is the union of commands 5 + 6 + 7. This represents everything that would appear in a PR opened right now.

**Edge cases:**
- If there is no diff at all (all three diffs are empty), print: "No changes found on this branch relative to `<base-branch>`. Nothing to describe." Then stop.
- If there are no commits but there are staged/unstaged changes, proceed using only those diffs.
- If on a detached HEAD, warn the user and attempt `git diff HEAD~1` as a fallback.

## Step 3: Find the PR template

Search for a PR template using this three-tier priority. **First match wins — stop searching.**

### Tier 1: Repo template

Using the repo root from Step 2, run a **single case-insensitive Glob** from `<repo-root>`:

```
**/*[Pp][Uu][Ll][Ll]_[Rr][Ee][Qq][Uu][Ee][Ss][Tt]_[Tt][Ee][Mm][Pp][Ll][Aa][Tt][Ee]*
```

This covers all casing variants (`pull_request_template.md`, `PULL_REQUEST_TEMPLATE.md`, etc.).

If multiple matches are found, prioritize by location:
1. Repo root (`<repo-root>/pull_request_template.md`)
2. `.github/` directory (`<repo-root>/.github/pull_request_template.md`)
3. `.github/PULL_REQUEST_TEMPLATE/` directory (`<repo-root>/.github/PULL_REQUEST_TEMPLATE/default.md`)
4. `docs/` directory (`<repo-root>/docs/pull_request_template.md`)

First match by priority wins. Ignore any matches inside `node_modules/`. Read the matched file with the Read tool.

### Tier 2: Global personal template

Check `~/.claude/templates/pr-description.md`. If it exists, read it with the Read tool.

### Tier 3: Hardcoded default

Use this structure:

```
## Summary
<2-3 sentence overview of what changed and why>

## Changes
- <grouped bullet points of key changes>

## Testing
- <how changes were tested or should be tested>
```

## Step 4: Generate the description

Using the template from Step 3, the full diff from Step 2, the commit messages, and the branch name, generate the PR description.

### Fill the template

1. **Parse** the template to identify section headings, checklists (`- [ ]` items), placeholder text, HTML comments, and fields (e.g., `Jira issue link: MX2-XXX`).
2. **Strip** all HTML comments (`<!-- ... -->`) and instructional placeholder text from the template.
3. **Fill** each section with real content derived from the diff and commits.

### Content guidelines

**Summary section:**
- 2-3 sentences maximum. State what changed and why.
- Infer "why" from commit messages and branch name.
- Do not pad with filler words or restate the obvious.

**Changes / details section:**
- Concise grouped bullet points.
- Group by theme or area (e.g., "Bug fixes", "Refactoring", "Tests"), not by individual file.
- Never enumerate every file changed — summarize at the concept level.
- With `--verbose`: mention specific key files and what changed in them.

**Ticket IDs:**
- Parse from branch name using pattern `[A-Z]+-\d+`.
- If none found in branch name, check commit messages.
- If still none found but the user passed a ticket ID as an argument (Step 1), use that.
- Fill into any ticket/issue link fields in the template.
- If no ticket ID from any source, leave the field with `(none)`.

**Checklists (auto-fill with confidence):**
- `[x]` items where you have **high confidence** from the diff:
  - Tests: checked if test files (`.test.`, `.spec.`, `_test.`, `test_`) appear in the diff
  - Documentation: checked if `.md` files or significant docstring changes appear in the diff
  - Atomic change: checked if the diff is focused on a single concern (use judgment)
- `[ ]` items where you **cannot verify** from the diff alone:
  - Linting/type checking passed (the skill does not run linters)
  - Technical debt assessment
  - Any item requiring subjective judgment you're unsure about
- If the template has the final "I've written a PR description" type item, always check it `[x]`.

**Unknown template sections:**
- If the template has sections the skill can't confidently fill, leave a brief `<!-- TODO: fill this in -->` marker.

### Output length targets

The description should be **readable in under 60 seconds**:
- Small PRs (1-5 files): ~100-150 words of prose
- Medium PRs (5-15 files): ~150-250 words of prose
- Large PRs (15+ files): ~250-350 words max of prose

Checklists do not count toward these targets. With `--brief`, aim for the low end. With `--verbose`, you may exceed the high end slightly.

## Step 5: Print the output

Print the generated PR description as **raw markdown** directly to the terminal.

**Critical output rules:**
- You MUST wrap the entire output in a markdown code fence (` ```markdown ``` `) so that raw markdown characters like `#` headings and `- [ ]` checkboxes are preserved literally in the terminal and not rendered by Claude Code. The user needs to see and copy the raw markdown syntax.
- Do NOT add any preamble like "Here's your PR description:" before the code fence.
- Do NOT add any postscript like "Copy this into GitHub" after the code fence.
- The content inside the code fence IS the PR description. Nothing else outside it.
- The user will copy the content from inside the fence and paste it directly into GitHub's PR description input field.
