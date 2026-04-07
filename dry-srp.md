Check the git diff between the current branch (both committed and uncommitted changes) against the default branch for this repository.

## Step 1: Identify changed files

**You MUST run the git diff command fresh right now** — do NOT reuse any previously observed or cached list of changed files from earlier in this conversation. From the fresh diff output, identify every file that was touched. Include both source files and test files.

## Step 2: Evaluate DRY violations

For each changed file, evaluate for DRY (Don't Repeat Yourself) violations. Look for:

- Duplicated code blocks or logic across changed files
- Repeated patterns that could be extracted into shared utilities or helper functions
- Copy-pasted code with minor variations
- Repeated string literals, magic numbers, or configuration values that should be constants
- Duplicated type definitions or interfaces across files

Classify each violation by severity:
- **Critical**: Exact or near-exact duplication of 10+ lines across multiple files
- **High**: Repeated logic patterns (3+ occurrences) that should be abstracted
- **Medium**: Duplicated smaller blocks (3-10 lines) or repeated inline expressions
- **Low**: Minor repetition such as repeated magic numbers or string literals

## Step 3: Evaluate SRP violations

For each changed file, evaluate for SRP (Single Responsibility Principle) violations. Look for:

- Functions or methods doing more than one thing (multiple unrelated side effects, mixed concerns)
- Classes or modules handling multiple unrelated responsibilities
- Files that mix concerns (e.g., business logic with I/O, data fetching with rendering, validation with transformation)
- Functions that are excessively long due to handling multiple responsibilities
- God objects or utility dumping grounds

Classify each violation by severity:
- **Critical**: A single function/class handling 3+ distinct responsibilities
- **High**: Mixed concerns that will cause issues during testing or maintenance (e.g., business logic tightly coupled with I/O)
- **Medium**: Functions doing two related but separable things
- **Low**: Minor concern mixing that could improve readability if separated

## Step 4: Refactor critical and high priority violations

Automatically refactor all **critical** and **high** severity DRY and SRP violations found in Steps 2 and 3. When refactoring:

- Extract duplicated code into shared utility functions or modules
- Split multi-responsibility functions into focused, single-purpose functions
- Separate mixed concerns into distinct modules or layers
- Ensure extracted code is properly exported and all call sites are updated
- Preserve existing behavior — refactoring must not change functionality

## Step 5: Review medium and low priority violations

Present all **medium** and **low** severity violations to the user in a clear summary, grouped by file, with:

- The violation type (DRY or SRP)
- The severity (medium or low)
- A brief description of the violation
- The file and line numbers involved
- A suggested refactoring approach

Ask the user which (if any) of these they would like refactored, then proceed with the ones they approve.

## Step 6: TypeScript strict typing

If any of the changed or newly created files are written in TypeScript, ensure that the `any` type is NEVER used. Replace any occurrences of `any` with strict, specific types — use the project's existing types, generics, `unknown`, or create precise type definitions as needed. This applies to both source files and test files.

## Step 7: Run the test suite

After all refactoring is complete, run the project's test suite to confirm all existing tests still pass. If any tests fail due to the refactoring, fix them. Do not finish until all tests pass.
