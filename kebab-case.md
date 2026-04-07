Enforce kebab-case file naming on all source files touched in the current branch's git diff. Rename violating files and directories, update all imports, jest.mock paths, and barrel re-exports.

## Step 0: Parse arguments

Check if the user passed any arguments:

- **`--base <branch>`**: Use `<branch>` as the base for the diff instead of the auto-detected default branch. The next argument after `--base` is the branch name.

**Determine the base branch:**
- If `--base <branch>` was provided, use that value directly.
- Otherwise, auto-detect the default branch by running `git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@'` (if this fails, use `main`).

Use the resulting base branch for all diff commands below.

## Step 1: Identify changed files

**You MUST run the git diff command fresh right now** — do NOT reuse any previously observed or cached list of changed files from earlier in this conversation.

Run two diff commands and combine the results:
1. `git diff --name-only --diff-filter=d <base>...HEAD` — committed changes on the current branch
2. `git diff --name-only --diff-filter=d` — uncommitted changes (staged and unstaged)

Combine and deduplicate the results. Filter to **source files only** — keep only files with these extensions: `.ts`, `.tsx`, `.js`, `.jsx`, `.css`, `.scss`.

Additionally, for each matched source file, check if an associated test file exists alongside it (e.g., `MyComponent.test.tsx`, `MyComponent.spec.ts`, `__tests__/MyComponent.test.tsx`). If so, include the test file in the list even if it was not in the diff — it will need to be renamed to match.

If no files remain after filtering, report that there are no source files in the diff and stop.

## Step 2: Identify kebab-case violations

For each file path from Step 1, check **every segment** of the path — both directory names and the filename — against the kebab-case rule.

**Kebab-case rule:** A name is valid kebab-case if it matches the regex `/^[a-z][a-z0-9]*(-[a-z0-9]+)*$/`.

**For filenames**, extract the base name by stripping all extensions. Handle compound extensions correctly:
- `MyComponent.test.tsx` → base name is `MyComponent`, extensions are `.test.tsx`
- `userAPI.spec.ts` → base name is `userAPI`, extensions are `.spec.ts`
- `styles.module.css` → base name is `styles`, extensions are `.module.css`
- `index.ts` → base name is `index`, extension is `.ts`

Only the **base name** is validated against the kebab-case rule. Extensions are preserved as-is.

**For directory names**, validate each directory segment individually.

**Kebab-case conversion logic** — convert the base name (or directory name) as follows:
1. Insert a hyphen before each uppercase letter that follows a lowercase letter or digit (camelCase/PascalCase boundaries): `myComponent` → `my-Component`
2. Insert a hyphen between consecutive uppercase letters followed by a lowercase letter (acronym boundaries): `XMLParser` → `XML-Parser`, `userAPI` → `user-API`
3. Replace underscores and spaces with hyphens: `my_file` → `my-file`
4. Lowercase everything: `my-Component` → `my-component`
5. Collapse multiple consecutive hyphens into one: `my--file` → `my-file`

**Examples:**
- `MyComponent` → `my-component`
- `userAPI` → `user-api`
- `XMLParser` → `xml-parser`
- `my_file` → `my-file`
- `myComponent.test.tsx` → `my-component.test.tsx`
- `already-kebab` → no change (skip)

Compute the full proposed path for each file. If a file is already fully kebab-case (all segments valid), skip it.

Collect all unique directory renames needed (deduplicating directories shared by multiple files).

## Step 3: Present proposed renames for approval

Present the proposed renames in two groups:

**Directory renames:**
| Current Directory | Proposed Directory |
|---|---|
| `src/MyComponent/` | `src/my-component/` |

**File renames:**
| Current Path | Proposed Path |
|---|---|
| `src/my-component/MyFile.tsx` | `src/my-component/my-file.tsx` |
| `src/my-component/MyFile.test.tsx` | `src/my-component/my-file.test.tsx` |

Note: File paths shown should reflect directory renames already applied.

Ask the user to approve the renames. Allow them to exclude specific items if needed. Do NOT proceed until the user confirms.

## Step 4: Execute renames with `git mv`

Execute the approved renames using `git mv` to preserve git history tracking.

**Order of operations:**
1. **Rename directories first**, processing from **deepest to shallowest** (longest path first). This prevents a parent directory rename from invalidating child paths before they are processed.
2. **Then rename files** within the (now potentially renamed) directories.

**Case-insensitive filesystem handling (macOS):**
On macOS (and other case-insensitive filesystems), `git mv Foo.ts foo.ts` will fail because the OS considers them the same file. For any rename where the only difference is letter casing, use a **two-step rename** through a temporary name:
```
git mv Foo.ts foo--kebab-tmp.ts && git mv foo--kebab-tmp.ts foo.ts
```
Apply the same two-step approach for directory renames that only change case.

**Always use the two-step approach to be safe** — it works correctly on both case-sensitive and case-insensitive filesystems.

After all renames are complete, run `git status` to confirm the renames were applied correctly.

## Step 5: Update all references across the repo

For each renamed file or directory, search the **entire repository** (not just diff files) for references to the old path and update them to the new path.

**What to update:**

1. **Import statements** — `import ... from '...'` and `import '...'` (both with and without file extensions in the import path)
2. **Require statements** — `require('...')` and `require("...")`
3. **jest.mock()** — `jest.mock('...')` and `jest.mock("...")`
4. **jest.requireActual()** — `jest.requireActual('...')` and `jest.requireActual("...")`
5. **Barrel file re-exports** — `export { ... } from '...'` and `export * from '...'` in index.ts/index.js files
6. **Dynamic imports** — `import('...')` expressions

**Matching strategy:**

For each renamed file, derive the old import-style reference — this is the path as it would appear in an import statement (typically without extension, using relative or aliased paths). Search for this string across all `.ts`, `.tsx`, `.js`, `.jsx` files in the repo.

Be careful with partial matches: `./MyComponent` should NOT match `./MyComponentWrapper`. Match the **full path segment** by ensuring the match is followed by a quote character (`'`, `"`, `` ` ``), a path separator, or end of string.

For directory renames, update every import that includes the old directory name as a path segment.

**After updating references**, present a summary of all files modified and the changes made.

## Step 6: Verify

Run verification to catch any broken references:

1. If a `tsconfig.json` exists, run `npx tsc --noEmit` (or the project's equivalent TypeScript check command) to verify no import errors.
2. If a linter is configured (look for `.eslintrc*`, `eslint.config.*`, or a `lint` script in `package.json`), run it.
3. If neither is available, do a final grep across the repo for any remaining references to the old file/directory names and report them.

If any broken references are found, fix them. Report a final summary of all changes made:
- Files/directories renamed
- Import references updated (count and files affected)
- Any issues that could not be auto-resolved
