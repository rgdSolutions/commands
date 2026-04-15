---
description: Produce concise QA testing steps for changes in the current branch's git diff
allowed-tools: Bash(git:*), Read, Glob, Grep
---

# Generate QA Testing Steps

Produce concise, browser-testable QA steps from the current branch's git diff. Output raw markdown for copy-paste into a ticket or message.

**You MUST run all git commands fresh right now.** Do NOT reuse any previously observed or cached diff, commit log, or branch info from earlier in this conversation.

## Step 0: Parse arguments

Check if the user passed any arguments:

- **`--base <branch>`**: Use `<branch>` as the base for the diff instead of the auto-detected default branch. The next argument after `--base` is the branch name.

**Determine the base branch:**
- If `--base <branch>` was provided, use that value directly.
- Otherwise, auto-detect the default branch by running `git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@'` (if this fails, use `main`).

Use the resulting base branch for all diff commands below.

## Step 1: Gather the diff

Run these commands **in parallel** to gather fresh context:

1. `git branch --show-current` — current branch name
2. `git diff <base>...HEAD --name-only` — committed changed files
3. `git diff --name-only` — unstaged changed files
4. `git diff --cached --name-only` — staged changed files
5. `git diff <base>...HEAD` — full committed diff
6. `git diff` — full unstaged diff
7. `git diff --cached` — full staged diff

The **full diff** is the union of commands 5 + 6 + 7. The **full file list** is the deduplicated union of commands 2 + 3 + 4.

**Edge cases:**
- If there is no diff at all (all three diffs are empty), print: "No changes found on this branch relative to `<base>`. Nothing to generate." Then stop.
- If there are no commits but there are staged/unstaged changes, proceed using only those diffs.

## Step 2: Analyze the diff for testable changes

From the full diff, identify every change that a human can test in a web browser. Focus on:

- **UI changes**: new or modified components, pages, forms, modals, dialogs, buttons, layout changes, styling changes
- **Logic changes**: validation rules, form behavior, conditional rendering, error handling, state changes visible to the user
- **Route/navigation changes**: new pages, redirected routes, changed URLs
- **API-driven behavior**: changes to what data is displayed, how data is fetched, loading/error states

**Ignore** changes that are not browser-testable: test files, build config, CI/CD, type definitions only, backend-only logic with no UI surface, documentation, lockfiles.

**Infer affected routes/URLs** by scanning the diff for:
- Route definitions (e.g., `path: '/settings'`, `<Route path="..."`, router config)
- Page/view component file paths (e.g., `pages/settings/profile.tsx` implies `/settings/profile`)
- Navigation calls (e.g., `navigate('/dashboard')`, `href="/..."`, `router.push(...)`)

**Classify changes**: determine whether the changes span multiple unrelated areas or are cohesive to a single feature.

## Step 3: Generate QA testing steps

Generate concise QA testing steps following these rules:

### Structure

- **If changes span multiple unrelated areas**: group steps under `##` headings named after each feature/area (e.g., `## User Profile Changes`, `## Dashboard Filters`)
- **If changes are cohesive**: use a single numbered list with no section headings

### Preconditions

If the diff implies specific setup is needed, include a `## Preconditions` section at the top. Examples of preconditions:
- Required user role or login state (e.g., "Log in as an admin user")
- Feature flags that must be enabled
- Test data that must exist
- Specific browser or device requirements

Only include this section when the diff clearly implies it. Do not add generic preconditions.

### Writing steps

- Each step is **one sentence**, action-oriented. Start with a verb: "Click...", "Navigate to...", "Enter...", "Select...", "Scroll to..."
- Use plain language a non-technical person can follow. Never reference code internals — no component names, function names, variable names, or CSS classes.
- Include inferred routes/URLs when available (e.g., "Navigate to `/settings/profile`")
- Include **expected results** only for key assertions — moments where a specific visual or behavioral change should be verified. Format: `→ Expected: [what should happen]`
- Do NOT add expected results to every step. Only add them where there is something specific and important to verify.
- Include **1-2 edge cases** per feature area when they are obvious from the diff (e.g., empty form submissions, invalid input, boundary values). Label them clearly: "Edge case: ..."
- Do NOT invent edge cases that are not suggested by the diff. Only include edge cases where the diff shows handling (or lack of handling) for those scenarios.

### Length targets

Keep the output short and scannable. QA testers will not read a wall of text.

- **Total steps**: 5-15 across the entire output
- **Per section**: 3-5 steps maximum when using grouped structure
- If the diff is large, prioritize the most important user-facing changes. You do not need to cover every single changed file.

## Step 4: Print the output

Print the generated QA steps as **raw markdown** directly to the terminal.

**Critical output rules:**
- Wrap the entire output in a markdown code fence (` ```markdown ``` `) so that raw markdown is preserved literally in the terminal.
- Do NOT add any preamble like "Here are your QA steps:" before the code fence.
- Do NOT add any postscript after the code fence.
- The content inside the code fence IS the QA testing steps. Nothing else outside it.
