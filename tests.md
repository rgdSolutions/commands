Check the git diff between the current branch (both committed and uncommitted changes) against a base branch for this repository.

## Step 0: Parse arguments

Check if the user passed any arguments:

- **`--base <branch>`**: Use `<branch>` as the base for the diff instead of the auto-detected default branch. The next argument after `--base` is the branch name.

**Determine the base branch:**
- If `--base <branch>` was provided, use that value directly.
- Otherwise, auto-detect the default branch by running `git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@'` (if this fails, use `main`).

Use the resulting base branch for all diff commands below.

## Step 1: Identify changed non-test files

**You MUST run the git diff command fresh right now** — do NOT reuse any previously observed or cached list of changed files from earlier in this conversation. Diff against the base branch determined in Step 0. From the fresh diff output, identify every file that was touched that is NOT a test file. Test files are files whose name contains `.test.`, `.spec.`, `_test.`, or `_spec.`, or that live inside a `__tests__/`, `tests/`, or `test/` directory.

## Step 2: Ensure associated test files exist

For each non-test file identified in Step 1, check whether an associated test file already exists in the appropriate location for this project's conventions. Determine the project's test file conventions by examining existing test files in the repository (co-located next to source files, in a `__tests__/` directory, in a top-level `tests/` directory, etc.) and follow that same pattern.

If the associated test file does not exist, create it with the correct imports and boilerplate for the project's test framework.

## Step 3: Ensure comprehensive test coverage

For each non-test file from Step 1, ensure it has comprehensive unit test coverage targeting 100% statement / 100% branch / 100% function / 100% line coverage. Review the source file thoroughly and add sufficient unit tests in the associated test file to cover:

- All exported functions and methods
- All branches (if/else, switch cases, ternary expressions, optional chaining fallbacks)
- All edge cases (null/undefined inputs, empty collections, boundary values)
- All error paths (thrown exceptions, rejected promises, error callbacks)
- All conditional logic and guard clauses

## Step 4: TypeScript strict typing

If any of the test files or source files are written in TypeScript, you must NEVER use the `any` type anywhere in the test files. Use strict, specific types instead — leverage the project's existing types, generics, `unknown`, or create precise type definitions as needed.

## Step 5: Audit pragma directives in modified files

Scan ALL files that appear in the git diff (both source and test files) for pragma directives that bypass linters or type checkers. Look for: `@ts-ignore`, `@ts-expect-error`, `@ts-nocheck`, `eslint-disable` (including `-next-line` and `-line` variants), `prettier-ignore`, and `istanbul ignore` / `c8 ignore`.

This covers every pragma in each modified file — not just newly added lines.

For each pragma found, classify it:

**Fixable** — the pragma suppresses an issue resolvable with proper typing, minor refactoring, or configuration. Examples:
- `@ts-expect-error` on a mock → use `vi.mocked()` or proper mock typing
- `eslint-disable @typescript-eslint/no-unused-vars` on destructuring → use `_` prefix or rest syntax
- `@ts-ignore` where a type assertion, generic, or interface would work

**Not fixable** — the pragma exists for a legitimate reason that cannot be resolved without disproportionate effort. Examples:
- `@ts-nocheck` on third-party/vendor code (e.g., Pendo snippets)
- `@ts-expect-error` on polyfills for features not yet in TypeScript's lib
- `@ts-expect-error` used intentionally in tests to simulate invalid states (e.g., `delete global.window`)
- `eslint-disable` on config files that must use CommonJS `require()`

**For fixable pragmas**: Present each one to the user with the file path, line number, the pragma, and the proposed fix. Ask the user for approval before making changes.

**For non-fixable pragmas**: Briefly list them with a short reason why they cannot be easily removed. Do not prompt for action.

If no pragma directives are found, skip this step silently.

## Step 6: Run the tests

After writing all tests and applying any pragma fixes, run the project's test suite to confirm all new and existing tests pass. Fix any failures before finishing.
