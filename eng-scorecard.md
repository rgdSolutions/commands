---
name: eng-scorecard
description: |
  Use when evaluating contributor code quality, analyzing engineering practices
  by author, reviewing team standards adherence, or auditing codebase health per
  engineer. Triggers on "analyze contributors", "evaluate engineers", "code
  quality by author", "team scorecard", "who writes the best code".
allowed-tools:
  - Bash
  - Agent
  - Read
  - Grep
  - Glob
---

# Engineering Scorecard

Announce: "Using eng-scorecard to analyze contributor code quality across this repository."

Evaluate every contributor in a git repository across 7 quality dimensions using git history. Produce a scored, tiered report that surfaces who strengthens codebase standards and who introduces debt.

## When to Use

- Evaluating engineering team quality patterns
- Auditing codebase health per contributor
- Reviewing standards adherence across a team
- Comparing code quality across contributors

## When NOT to Use

- Reviewing a single PR (use code-review skills)
- Analyzing code quality without per-author breakdown
- The working directory is not a git repository

## Invocation

- `/eng-scorecard` — analyze the full repository
- `/eng-scorecard <path>` — scope analysis to a subdirectory (e.g., `/eng-scorecard src/typescript/`)

If a path argument is provided, ALL git commands must include `-- <path>` to scope results.

## Dimensions & Weights

| # | Dimension | Weight | Measures |
|---|---|---|---|
| 1 | **Test Culture** | 25% | % of changesets touching test files; net test line additions |
| 2 | **Type Safety** | 20% | Net `typing.Any` / `: any` introductions via tight regexes |
| 3 | **Commit Discipline** | 15% | Avg lines/changeset, avg files/changeset, ticket reference % |
| 4 | **Code Hygiene** | 15% | Net TODO/FIXME/HACK, net console.log/print introductions |
| 5 | **SRP Signals** | 10% | Generic file names introduced, functions 50+ lines (Python) / 40+ lines (TS) |
| 6 | **Pragma Hygiene** | 10% | Net lint suppression introductions (`type: ignore`, `noqa`, `eslint-disable`, etc.) |
| 7 | **Mock Discipline** | 5% | Net `unittest.mock` / `MagicMock` / `patch()` introductions (Python only) |

**Language awareness:** Detect which languages exist in scope. If no Python files exist, skip Mock Discipline and redistribute its 5% weight proportionally across the remaining 6 dimensions. If no TypeScript files exist, skip TS-specific signals within each dimension.

## Execution Flow

Three phases. Phase 1 is sequential. Phase 2 is parallel. Phase 3 synthesizes.

```
Phase 1: Author Discovery (sequential)
    |
    v
Phase 2: Launch 3 agents IN PARALLEL
    |-- Agent A: Test & Type Safety
    |-- Agent B: Commit, Hygiene & Pragmas
    |-- Agent C: SRP & Mock
    |
    v
Phase 3: Synthesis (main thread)
```

---

## Phase 1: Author Discovery

Run these commands to build the contributor list and context. All commands MUST include `--no-merges`.

### Step 1.1: Get all authors

```bash
git log --no-merges --format='%aN' -- <path_if_scoped> | sort | uniq -c | sort -rn
```

`%aN` respects `.mailmap` for identity normalization.

### Step 1.2: Case-insensitive deduplication

After getting the author list, merge authors whose names differ only by case. Sum their commit counts. Report merged identities in the output header.

### Step 1.3: Exclude bots

Remove authors matching (case-insensitive):
```
\[bot\]|dependabot|renovate|github-actions|semantic-release|snyk
```

### Step 1.4: Group commits into changesets

For each author, extract PR numbers from commit messages:
```bash
git log --no-merges --author="<name>" --format='%H %s' -- <path_if_scoped>
```

Extract `\(#(\d+)\)` from the subject line. Commits sharing the same PR number form one changeset. Commits without a PR number are standalone changesets.

### Step 1.5: Apply minimum threshold

Exclude authors with fewer than 20 changesets. Record excluded count for the header.

### Step 1.6: Detect languages

```bash
git log --no-merges --format=%H -- <path_if_scoped> '*.py' | head -1
git log --no-merges --format=%H -- <path_if_scoped> '*.ts' '*.tsx' | head -1
```

Set flags: `has_python`, `has_typescript`. These determine which dimension signals to collect.

### Step 1.7: Detect merge strategy

```bash
git log --merges --format=%H -- <path_if_scoped> | wc -l
git log --no-merges --format=%H -- <path_if_scoped> | wc -l
```

If merge commits are <5% of total, report "squash-merge detected" as a caveat.

### Step 1.8: Auto-detect ticket pattern

```bash
git log --no-merges --format='%s' -- <path_if_scoped> | head -100
```

Extract the most common `[A-Z]+-\d+` pattern (e.g., `MX2-`, `JIRA-`, `PROJ-`). Also check for patterns like `MX2LAW` followed by a number. If no dominant pattern appears in >20% of messages, skip ticket reference % from Commit Discipline scoring.

### Phase 1 Output

Provide ALL of the following to each parallel agent:
- Author list with changeset counts
- Changeset groupings (which commits belong to which PR)
- Language flags (`has_python`, `has_typescript`)
- Path scope (if any)
- Ticket pattern (if detected)

---

## Phase 2: Parallel Agent Dispatch

Launch ALL THREE agents in a SINGLE message with multiple Agent tool calls. Each agent receives the full Phase 1 output.

**CRITICAL:** Use `subagent_type: "general-purpose"` for all agents. Provide each agent with the complete regex patterns, counting methodology, and output format specified below. Each agent must return its results as a structured table.

### Agent A: Test & Type Safety (45% weight)

Prompt the agent with these exact instructions:

**Test Culture (25%):**
For each author, count:
1. Total changesets (provided in input)
2. Changesets that touch at least one test file

Test file patterns: files matching any of `**/test*`, `**/*test*`, `**/__tests__/**`, `**/*_test.*`, `**/*spec.*`

```bash
# For each author, count changesets touching test files
git log --no-merges --author="<name>" --format='%H %s' -- <path_if_scoped> | while read hash subject; do
  pr=$(echo "$subject" | grep -oP '\(#\K\d+(?=\))' || echo "$hash")
  if git diff-tree --no-commit-id -r "$hash" --name-only | grep -qiE '(test|__tests__|spec)'; then
    echo "$pr"
  fi
done | sort -u | wc -l
```

Metric: `test_pct = changesets_with_tests / total_changesets * 100`

**Type Safety (20%):**
For each author, compute NET introductions using `git log -p`:

Python `Any` (only if `has_python`):
```
ADDED:   grep -cP '^\+.*(?:typing\.Any|:\s*Any\b|\[Any\]|,\s*Any\b)'
REMOVED: grep -cP '^-.*(?:typing\.Any|:\s*Any\b|\[Any\]|,\s*Any\b)'
NET = ADDED - REMOVED
```

TypeScript `any` (only if `has_typescript`):
```
ADDED:   grep -cP '^\+.*(?::\s*any\b|as\s+any\b|<any>|any\[\]|any\s*\||\|\s*any\b)'
REMOVED: grep -cP '^-.*(?::\s*any\b|as\s+any\b|<any>|any\[\]|any\s*\||\|\s*any\b)'
NET = ADDED - REMOVED
```

**IMPORTANT:** These are NET counts. Negative values mean the author removed more than they added (good).

Agent A must return a table:
```
| Author | Changesets | Test Changesets | Test% | Net Any (py) | Net any (ts) | Net Any Total |
```

### Agent B: Commit, Hygiene & Pragmas (40% weight)

**Commit Discipline (15%):**
For each author, sample up to 50 recent changesets:

```bash
# Average lines changed per changeset (exclude vendor files)
git log --no-merges --author="<name>" --format=%H -- <path_if_scoped> | head -50 | while read sha; do
  git diff-tree --numstat --no-commit-id "$sha" | grep -vE '(package-lock\.json|pnpm-lock\.yaml|yarn\.lock|\.generated\.|\.pb\.go|\.min\.js|\.min\.css)' | awk '{a+=$1; d+=$2} END {print a, d}'
done
```

Compute: avg additions, avg deletions, avg files per changeset.

Ticket reference %: count changesets whose commits contain the detected ticket pattern / total changesets. If no ticket pattern was detected, skip this sub-metric.

**Code Hygiene (15%):**
For each author, compute NET introductions. Exclude commits matching `^Deploy\b|^Revert "` from pattern analysis.

```bash
git log --no-merges --author="<name>" -p -- <path_if_scoped> | grep -v '^commit\|^Author\|^Date'
```

TODO/FIXME/HACK (net):
```
ADDED:   grep -ciP '^\+.*\b(?:TODO|FIXME|HACK|XXX)\b'
REMOVED: grep -ciP '^-.*\b(?:TODO|FIXME|HACK|XXX)\b'
```

console.log (net, TS only):
```
ADDED:   grep -cP '^\+.*console\.log'
REMOVED: grep -cP '^-.*console\.log'
```

print() (net, Python only):
```
ADDED:   grep -cP '^\+.*\bprint\('
REMOVED: grep -cP '^-.*\bprint\('
```

**Pragma Hygiene (10%):**
Net lint suppression introductions:

Python (if `has_python`):
```
ADDED:   grep -cP '^\+.*(?:# type: ignore|# noqa|# pylint: disable)'
REMOVED: grep -cP '^-.*(?:# type: ignore|# noqa|# pylint: disable)'
```

TypeScript (if `has_typescript`):
```
ADDED:   grep -cP '^\+.*(?:eslint-disable|@ts-ignore|@ts-expect-error)'
REMOVED: grep -cP '^-.*(?:eslint-disable|@ts-ignore|@ts-expect-error)'
```

Agent B must return a table:
```
| Author | Avg +/- | Files/cs | Ticket% | Net TODO | Net console.log | Net print | Net Pragmas |
```

### Agent C: SRP & Mock (15% weight)

**SRP Signals (10%):**
For each author, check for:

1. Generic file names introduced — count new files (in `^\+\+\+ b/` diff headers) with path segments matching:
```
\b(?:utils|helpers|common|shared|misc)\b
```

2. Long functions added — in `git log -p` output for each author, find function definitions in added lines and estimate length:
   - Python: `^\+\s*def ` openings — count added lines until next `^\+\s*def `, `^\+\s*class `, or 100 lines (whichever comes first). Flag functions with 50+ added lines.
   - TypeScript: `^\+.*(function |const \w+ = |=> \{)` openings — similar heuristic with 40-line threshold.

This is an approximation. Report it as such.

**Mock Discipline (5%, Python only — skip if not `has_python`):**
Net `unittest.mock` introductions:
```
ADDED:   grep -cP '^\+.*(?:from unittest\.mock|from unittest import mock|MagicMock|create_autospec|mocker\.patch|@patch)'
REMOVED: grep -cP '^-.*(?:from unittest\.mock|from unittest import mock|MagicMock|create_autospec|mocker\.patch|@patch)'
```

Agent C must return a table:
```
| Author | Generic Files | Long Functions | SRP Total | Net Mocks |
```

---

## Phase 3: Synthesis

After all three agents return, synthesize the results.

### Step 3.1: Combine raw data

Merge the three agent tables into a single raw data table with all metrics per author.

### Step 3.2: Determine language stack per author

For each author, check which file types they primarily touch:
```bash
git log --no-merges --author="<name>" --format=%H -- <path_if_scoped> '*.py' | wc -l  # python commits
git log --no-merges --author="<name>" --format=%H -- <path_if_scoped> '*.ts' '*.tsx' | wc -l  # ts commits
git log --no-merges --author="<name>" --format=%H -- <path_if_scoped> '*.tf' '*.hcl' | wc -l  # infra commits
```

Classify as: Python, TypeScript, Infra, Full-stack, or Mixed based on dominant stack (>60% = primary, otherwise mixed).

### Step 3.3: Normalize scores

For each dimension, normalize to 0-10 scale relative to the cohort:

```
For "higher is better" metrics (test%, ticket%):
  score = (value - min) / (max - min) * 10

For "lower is better" metrics (net Any, net TODO, net pragmas, net mocks, SRP flags, avg commit size):
  score = (max - value) / (max - min) * 10
```

If all values are identical for a dimension, all engineers get 5.0.

### Step 3.4: Compute weighted overall

```
overall = test_score * 0.25 + type_score * 0.20 + commit_score * 0.15 + hygiene_score * 0.15 + srp_score * 0.10 + pragma_score * 0.10 + mock_score * 0.05
```

If Mock Discipline was skipped (no Python), redistribute 5% proportionally:
```
overall = test_score * 0.2632 + type_score * 0.2105 + commit_score * 0.1579 + hygiene_score * 0.1579 + srp_score * 0.1053 + pragma_score * 0.1053
```

### Step 3.5: Assign tiers

Sort engineers by overall score descending.

- **Tier 1** (top 25%): Strongest alignment with codebase standards
- **Tier 2** (middle 50%): Strong with notable trade-offs
- **Tier 3** (bottom 25%): Legacy patterns or high debt signals

For each engineer in the tier summary, write 1-2 sentences explaining **why** they landed in that tier — cite specific dimension strengths or weaknesses.

---

## Output Format

Present the report in this exact order:

### 1. Header

```markdown
## Engineering Scorecard -- {repo name} {path scope if provided}
Analyzed {N} contributors ({M} excluded: {X} bots, {Y} below 20-changeset threshold)
Merge strategy: {squash/merge/mixed}
Identity merges applied: {list, or "none"}
Languages detected: {Python, TypeScript, Terraform, etc.}
Ticket pattern: {pattern detected, or "none detected"}
```

### 2. Language Profile Table

| Engineer | Changesets | Python | TypeScript | Infra | Primary Stack |

### 3. Raw Data Table

| Engineer | Changesets | Stack | Tests% | Net Any | Avg +/- | Files/cs | Ticket% | Net TODO | Net Pragmas | Net Mocks | SRP flags |

### 4. Per-Dimension Highlights

For each of the 7 dimensions, print a **brief definition** of the dimension followed by the **top 3** and **bottom 3** engineers. The definition helps readers who aren't familiar with the metric understand what they're looking at.

Format:

> **Test Culture** (25%) -- Do they ship tests with their code? Measures % of changesets that include test file changes.
> Top: Engineer A (72%), Engineer B (78%), Engineer C (65%)
> Bottom: Engineer D (8%), Engineer E (7%), Engineer F (6%)
>
> **Type Safety** (20%) -- Do they strengthen or weaken the type system? Measures net `typing.Any` (Python) and `: any` (TypeScript) introductions. Negative = removed more than added (good).
> Top: ...
> Bottom: ...
>
> **Commit Discipline** (15%) -- Are changesets small, focused, and traceable? Measures avg lines changed, avg files per changeset, and ticket reference %.
> Top: ...
> Bottom: ...
>
> **Code Hygiene** (15%) -- Do they leave the codebase cleaner than they found it? Measures net TODO/FIXME/HACK markers, console.log, and print() introductions. Negative = cleaned up more than added (good).
> Top: ...
> Bottom: ...
>
> **SRP Signals** (10%) -- Do they respect single-responsibility? Measures generic file names introduced (utils/helpers/common/shared/misc) and functions exceeding length thresholds. Lower is better.
> Top: ...
> Bottom: ...
>
> **Pragma Hygiene** (10%) -- Do they suppress linters or fix the underlying issue? Measures net lint suppression introductions (`# type: ignore`, `# noqa`, `eslint-disable`, etc.). Negative = removed more suppressions than added (good).
> Top: ...
> Bottom: ...
>
> **Mock Discipline** (5%, Python only) -- Do they test outcomes or test wiring? Measures net `unittest.mock`/`MagicMock`/`patch()` introductions. Negative = removed legacy mocks (good).
> Top: ...
> Bottom: ...

### 5. Scored Scorecard

| Engineer | Test (25%) | Type (20%) | Commit (15%) | Hygiene (15%) | SRP (10%) | Pragma (10%) | Mock (5%) | **Overall** |

Each cell shows the 0.0-10.0 normalized score, rounded to 1 decimal.

### 6. Tier Summary

For each tier:
```markdown
**Tier 1 -- Strongest alignment with standards:**
- **Engineer Name** -- [1-2 sentence explanation citing specific strengths]

**Tier 2 -- Strong with trade-offs:**
- **Engineer Name** -- [1-2 sentence explanation citing strengths AND specific trade-offs]

**Tier 3 -- Legacy patterns / high debt signals:**
- **Engineer Name** -- [1-2 sentence explanation citing specific concerns]
```

### 7. Caveats

Auto-generate based on what was detected:
- If squash-merge: "Squash-merge detected. PR authors get sole credit for all co-authored work in squashed commits."
- If dimensions skipped: "Mock Discipline skipped (no Python in scope). Weight redistributed."
- If identity merges: "The following identities were merged: ..."
- Always include: "SRP function-length detection is heuristic (not AST-based). DRY violations are not measurable from git history alone."
- Always include: "Net counting rewards cleanup work. Negative values mean the engineer removed more debt than they introduced."

---

## Hard Rules

These are NON-NEGOTIABLE. Every agent must follow them.

1. **`--no-merges` on ALL git commands.** Merge commits attribute others' work to the merger and inflate every metric.
2. **Use `%aN` (not `%an`) in ALL git log format strings.** `%aN` respects `.mailmap` for identity normalization.
3. **Net counting for ALL pattern-based dimensions.** Count `^\+` matches MINUS `^\-` matches. A positive net means debt introduced. A negative net means debt cleaned up. NEVER count additions only.
4. **Exclude vendor/generated files from line counts.** Files matching `package-lock.json|pnpm-lock.yaml|yarn.lock|*.generated.*|*.pb.go|*.min.js|*.min.css` must be excluded from commit discipline calculations.
5. **Exclude deploy/revert commits from pattern analysis.** Commits whose subject matches `^Deploy\b|^Revert "` are excluded from TODO/Any/pragma/mock pattern counting (but kept in total changeset counts).
6. **Report primary language stack per engineer.** This contextualizes scores — an infra-heavy engineer will naturally score differently on type safety than a frontend engineer.
7. **Scope ALL commands with `-- <path>` when a path argument is provided.** Never analyze outside the requested scope.

## Edge Cases

| Edge Case | Handling |
|---|---|
| Merge commits double-count | `--no-merges` on all git commands |
| Duplicate git identities | `.mailmap` via `%aN` + case-insensitive grouping |
| English "any" in comments/strings | Tight regex: `: any`, `as any`, `<any>`, `any[]` only |
| Infra-heavy engineers | Report stack; skip/redistribute inapplicable dimensions |
| Cleanup work penalized | Net counting (additions minus removals) |
| Per-commit splits PR work | Group by PR number `(#NNNN)` from commit message |
| Deploy/revert commits | Exclude from pattern analysis |
| Vendor/lockfiles | Exclude from line counts |
| Bot accounts | Auto-exclude matching patterns |
| Low sample size | 20 changeset minimum threshold |
| Squash-merge attribution | Detect and caveat in output |
| Initial scaffolding commits | Cohort normalization dampens outlier effect |
| Co-authored commits | Attributed to commit author only |
| Repo with no `.mailmap` | Case-insensitive grouping fallback |
| No dominant ticket pattern | Skip ticket % from Commit Discipline |
| All values identical for a dimension | All engineers score 5.0 |

## Common Mistakes

| Mistake | Prevention |
|---|---|
| Forgetting `--no-merges` on a git command | Hard rule #1. Every single git log/diff command. No exceptions. |
| Using `%an` instead of `%aN` | Hard rule #2. `%aN` respects mailmap. |
| Counting only additions (not net) | Hard rule #3. Always subtract removals. |
| Matching English word "any" in TS | Use tight regex — `: any`, `as any`, `<any>`, `any[]`, `any\|`, `\|any` only. |
| Including lockfile diffs in commit size | Exclude via grep filter on file paths. |
| Running agents sequentially instead of parallel | Phase 2 agents MUST launch in a single message with multiple Agent tool calls. |
| Attributing mock usage in TS repos | Mock Discipline is Python-only. Skip if no Python. |
| Penalizing engineers who clean up debt | Net counting handles this — negative net = good. Verify your output shows this. |
| Not scoping git commands when path provided | Every git command must include `-- <path>` when a path argument was given. |
