#### Installation Instructions:

```bash
cd ~/.claude
```

```bash
git clone https://github.com/rgdSolutions/commands.git
```

#### Tips:
- Many of these commands use a `--base` parameter to explicitly set the base branch from which to calculate git DIFFs
- Many of these commands use the `--auto` parameter to run in "fire and forget" mode

#### Commands:

| Command | Summary |
|---|---|
| `/pr-comments` | Cycles through unresolved PR review comments, shows code context, and resolves each with a fix or dismissal. Can run fully autonomously with the `--auto` parameter. |
| `/pr-description` | Generates a ready-to-paste PR description from your branch diff, using the repo's PR template if available. |
| `/pr-chunking` | Splits a large branch into smaller, sequential stacked PRs for easier code review. |
| `/tests` | Detects changed files missing test coverage and generates unit tests targeting 100% coverage using your repo's existing test conventions. |
| `/dry-srp` | Audits your diff for DRY and Single Responsibility Principle violations, classifying issues by severity. |
| `/remove-dead-code` | Finds and removes unused imports, functions, variables, and unreachable code introduced on your branch. |
| `/kebab-case` | Enforces kebab-case file naming across your diff, renaming files and updating all imports and references automatically. |
| `/astro-tokens` | Audits the diff for hardcoded colors and spacing that should use design system tokens (astro-code specific). |
| `/mx2-tokens` | Audits the diff for hardcoded colors, spacing, and borderRadius that should use MX2 design tokens (docr specific). Supports `--auto` for fire-and-forget mode. |