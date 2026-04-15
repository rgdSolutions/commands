Audit the git diff for hardcoded color, spacing, and borderRadius styles that should use MX2 design tokens instead. This command is exclusively for the docr project.

## Autonomous Mode (`--auto`)

When the `--auto` flag is passed (e.g., `/mx2-tokens --auto`), this command MUST run fully autonomously without any user interaction:

- **Do NOT use `AskUserQuestion` at any point.** Skip every prompt that would normally ask the user to choose.
- **Auto-fix all violations** that have an exact or approximate token match. Skip violations with no matching token.
- **The goal is zero user prompts.** The user should be able to run `/mx2-tokens --auto` and walk away while every violation is addressed.

**Important:** Autonomous mode is triggered **only** by the `--auto` flag, NOT by the session's permission mode.

## Step 0: Parse arguments

Check if the user passed any arguments:

- **`--base <branch>`**: Use `<branch>` as the base for the diff instead of the auto-detected default branch. The next argument after `--base` is the branch name.
- **`--auto`**: Enable autonomous mode (see above). Remove the flag from the argument list before proceeding.

**Determine the base branch:**
- If `--base <branch>` was provided, use that value directly.
- Otherwise, auto-detect the default branch by running `git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@'` (if this fails, use `main`).

Use the resulting base branch for all diff commands below.

## Step 1: Identify changed files

**You MUST run the git diff commands fresh right now** -- do NOT reuse any previously observed or cached list of changed files from earlier in this conversation.

Run all three commands and union the results:
- `git diff <base>...HEAD --name-only` (committed changes)
- `git diff --name-only` (unstaged changes)
- `git diff --cached --name-only` (staged changes)

From the combined list, keep only files with these extensions: `.tsx`, `.jsx`, `.ts`, `.js`.

**Exclude** any file whose path ends with `mx2-design-tokens.ts` -- it IS the token definitions and must not be linted.

If no applicable files were changed, report "No applicable files in diff" and stop.

## Step 2: Build the token lookup

Read the design token file: `src/typescript/mx2/common-ts/providers/mx2-design-tokens.ts`

Parse the token definitions and build lookup maps for the `light` mode values (the primary/default mode):

### Color tokens (hex value -> token path)

Extract all color values from the returned object. The token paths follow `theme.palette.mx2.colors.*` for usage. Key groups:

- **`colors.text`**: `primary`, `secondary`, `disabled`, and `states.*`
- **`colors.primary`**: `main`, `dark`, `light`, `contrast`, and `states.*`
- **`colors.secondary`**: `main`, `dark`, `light`, `contrast`, and `states.*`
- **`colors.tertiary`**: `main`, `dark`, `light`, `contrast`, and `states.*`
- **`colors.error`**: `main`, `dark`, `light`, `contrast`, and `states.*`
- **`colors.warning`**: `main`, `dark`, `light`, `contrast`, and `states.*`
- **`colors.success`**: `main`, `dark`, `light`, `contrast`, and `states.*`
- **`colors.action`**: `active`, `hover`, `selected`, `focus`, `disabled`, `disabledBackground`
- **`colors.common.blackStates`**: `main`, `hover`, `selected`, `focus`, `focusVisible`, `outlinedBorder`
- **`colors.common.whiteStates`**: `main`, `hover`, `selected`, `focus`, `focusVisible`, `outlinedBorder`

Also extract brand colors:
- **`mx2Colors.blue`**: `100` through `600`
- **`mx2Colors.yellow`**: `100`, `500`, `700`
- **`mx2Colors.neutral`**: `100` through `900`
- **`mx2Colors.shades`**: `white`, `black`

Resolve all references (e.g., `grey[900]` -> `#212121`, `withOpacity(...)` -> computed hex) to get the final hex value for each token.

### Spacing tokens (px value -> token key)

| Token key | Pixel value |
|---|---|
| `spacing['1']` | 8px |
| `spacing['2']` | 16px |
| `spacing['3']` | 24px |
| `spacing['4']` | 32px |
| `spacing['5']` | 40px |
| `spacing['6']` | 48px |
| `spacing['7']` | 56px |
| `spacing['8']` | 64px |
| `spacing['9']` | 72px |
| `spacing['10']` | 80px |
| `spacing['11']` | 88px |
| `spacing['12']` | 96px |

### Border radius tokens

| Token key | Value |
|---|---|
| `borderRadius.default` | 8px |
| `borderRadius.none` | 0px |
| `borderRadius.fullRound` | 100% |

### MUI color palette names (for import detection)

The 19 MUI palettes exported under `muiColors`: `amber`, `blue`, `blueGrey`, `brown`, `cyan`, `deepOrange`, `deepPurple`, `green`, `grey`, `indigo`, `lightBlue`, `lightGreen`, `lime`, `orange`, `pink`, `purple`, `red`, `teal`, `yellow`.

## Step 3: Scan for violations

For each file from Step 1, read it and scan for the following five categories of violations:

### Category A: Hardcoded colors in style contexts

Search for hardcoded hex colors (`#xxx`, `#xxxxxx`, `#xxxxxxxx`) and `rgb()`/`rgba()` values within:

- **`sx` prop objects**: Inside `sx={{`, `sx={[`, or `sx={(theme) =>`
- **Inline style objects**: Inside `style={{`
- **Styled-components / Emotion**: Inside `styled(...)({`, `styled.div({`, or tagged template literals `` styled(...)` `` / `` styled.div` ``

Flag when the value is assigned to color-related properties:
`color`, `backgroundColor`, `background`, `borderColor`, `fill`, `stroke`, `outlineColor`, `boxShadow`, `textDecorationColor`

### Category B: Hardcoded spacing in style contexts

Within the same style contexts as Category A, detect hardcoded pixel values or bare MUI spacing numbers for spacing properties:

**Shorthand props** (MUI `sx` only -- bare numbers are multipliers, value x 8px):
`p`, `pt`, `pr`, `pb`, `pl`, `px`, `py`, `m`, `mt`, `mr`, `mb`, `ml`, `mx`, `my`, `gap`

**Full property names** (all style contexts):
`padding`, `paddingTop`, `paddingRight`, `paddingBottom`, `paddingLeft`, `paddingX`, `paddingY`, `margin`, `marginTop`, `marginRight`, `marginBottom`, `marginLeft`, `marginX`, `marginY`, `gap`, `rowGap`, `columnGap`, `top`, `right`, `bottom`, `left`, `width`, `height`, `minWidth`, `maxWidth`, `minHeight`, `maxHeight`

**Both formats count as violations:**
- String px values: `padding: '16px'` or `padding: '16px 24px'`
- Bare numbers in non-shorthand properties: `padding: 16` (raw pixels, not MUI multiplier)

### Category C: Hardcoded borderRadius

Within the same style contexts, detect hardcoded `borderRadius` values:
- `borderRadius: 8` or `borderRadius: '8px'` -> should use token
- `borderRadius: 4` or `borderRadius: '4px'` -> no matching token, flag it

### Category D: Direct MUI color imports

Detect import statements that pull colors directly from `@mui/material/colors`:

```
import { grey, blue, ... } from '@mui/material/colors'
import red from '@mui/material/colors/red'
```

Then scan for usages of those imported identifiers (e.g., `grey[500]`, `blue[200]`). Each usage is a violation.

### Category E: Non-MX2 theme palette usage

Detect any `theme.palette.*` reference that does NOT go through `theme.palette.mx2.*`:

- `theme.palette.primary.main` -- violation (should be `theme.palette.mx2.colors.primary.main`)
- `theme.palette.grey[500]` -- violation
- `theme.palette.error.light` -- violation
- `theme.palette.text.secondary` -- violation
- `theme.palette.action.hover` -- violation

**NOT a violation:** `theme.palette.mx2.colors.primary.main` (already using MX2 tokens)

Also detect destructured palette access that bypasses MX2:
- `({ theme }) => theme.palette.primary.main` in styled-components
- `(theme) => theme.palette.error.dark` in sx callbacks

### Exclusions -- do NOT flag any of these:

- **Zero values**: `0`, `0px`, `'0px'`, `'0'`
- **CSS keywords**: `'transparent'`, `'inherit'`, `'currentColor'`, `'none'`, `'auto'`, `'initial'`, `'unset'`
- **Already tokenized (MX2)**: values using `theme.palette.mx2.*`, `theme.spacing(...)`, `theme.shape.borderRadius`, or `var(--mx2-*)`
- **Percentage / viewport units**: `'100%'`, `'50vh'`, `'100vw'`, etc.
- **`calc()` expressions**: too complex to auto-replace
- **Structural border widths**: `1px`, `2px` (not spacing tokens -- MX2 spacing starts at 8px)
- **Layout keywords**: `'fit-content'`, `'min-content'`, `'max-content'`
- **Negative values**: `-8px`, `-1`, etc.
- **Non-MUI palette theme references**: `theme.breakpoints`, `theme.typography`, `theme.transitions`, `theme.zIndex` (not color/spacing tokens)

## Step 4: Map violations to token replacements

### Colors: find the matching MX2 color token

1. Normalize the hardcoded value to a 6-digit lowercase hex string
2. Search the color lookup (from Step 2) for an exact hex match
3. If no exact match, find the nearest token by Euclidean RGB distance
4. Mark match quality: "exact" or "approximate (nearest: `token.path`)"

**Replacement format depends on context:**

| Context | Replacement |
|---|---|
| `sx` prop (no existing theme callback) | `(theme) => theme.palette.mx2.colors.X.Y` |
| `sx` prop (inside existing theme callback) | `theme.palette.mx2.colors.X.Y` |
| `styled` template literal | `${({ theme }) => theme.palette.mx2.colors.X.Y}` |
| `styled` object syntax (inside theme callback) | `theme.palette.mx2.colors.X.Y` |
| Inline `style={{}}` | `var(--mx2-colors-X-Y)` |

### Spacing: convert to MX2 spacing token

1. Determine the pixel value:
   - MUI `sx` shorthand props (`p`, `m`, `gap`, etc.) with bare number: multiply by 8
   - Explicit px string or bare number on full property names: use directly
2. Find `spacing[N]` where `N * 8 = px_value`
3. Replacement: `theme.palette.mx2.spacing['N']` (or appropriate format for context)
4. If px value is not a multiple of 8 or exceeds 96px: flag as "no matching token" with nearest options

### Border radius: map to MX2 borderRadius token

| Value | Token | Replacement |
|---|---|---|
| `8` or `'8px'` | `borderRadius.default` | `theme.palette.mx2.borderRadius.default` |
| `'100%'` | `borderRadius.fullRound` | `theme.palette.mx2.borderRadius.fullRound` |
| Other values | No match | Flag as "no matching token" |

### MUI color imports: replace with MX2 token

1. Look up the hex value from the MUI palette (e.g., `grey[500]` -> `#9e9e9e`)
2. Find the matching or nearest MX2 color token
3. Suggest replacing the usage with the MX2 token path
4. If the import has no remaining usages after fixes, suggest removing the import line

### Non-MX2 theme palette: replace path

For `theme.palette.X.Y` violations, suggest the MX2 equivalent:
- `theme.palette.primary.main` -> `theme.palette.mx2.colors.primary.main`
- `theme.palette.error.dark` -> `theme.palette.mx2.colors.error.dark`
- `theme.palette.text.secondary` -> `theme.palette.mx2.colors.text.secondary`
- `theme.palette.grey[500]` -> find the matching `mx2Colors.neutral` shade or `colors.text/action` token

## Step 5: Report violations

Present all violations grouped by file. For each violation, show:

```
path/to/file.tsx

  Line 42: backgroundColor: '#222222'
    -> theme.palette.mx2.colors.primary.main  (exact match)

  Line 58: padding: '16px'
    -> theme.palette.mx2.spacing['2']  (exact: spacing['2'] = 16px)

  Line 73: import { grey } from '@mui/material/colors' ... grey[500] on line 80
    -> theme.palette.mx2.colors.text.secondary  (approximate match)

  Line 91: borderRadius: 4
    -> no matching MX2 token (nearest: borderRadius.default = 8px)

  Line 105: theme.palette.primary.main
    -> theme.palette.mx2.colors.primary.main
```

At the end of the report, show a summary count:
- Total violations found
- Exact matches
- Approximate matches
- No token available

**Interactive mode (default):** Ask the user: **"Which violations should I fix? (all / specific files / specific items / none)"** -- wait for response before proceeding.

**Autonomous mode (`--auto`):** Skip the prompt. Auto-fix all violations that have an exact or approximate token match. Skip violations with no matching token.

## Step 6: Apply approved fixes

For each violation the user approved (or all fixable violations in `--auto` mode):

1. **Hardcoded color/spacing/borderRadius**: Replace the value with the token reference, using the correct syntax for the context (see Step 4 replacement table).

2. **MUI color import usages**: Replace each `colorName[shade]` usage with the MX2 token reference. After all usages in the file are replaced, remove the `@mui/material/colors` import line if no remaining usages exist.

3. **Non-MX2 theme palette**: Replace the `theme.palette.X.Y` path with `theme.palette.mx2.colors.X.Y`.

4. **Theme callback wrapping**: If a violation is inside an `sx` prop that does not already have a theme callback, wrap the property value in `(theme) => ...`. If the `sx` prop already has a theme callback, just replace the value directly.

Apply all edits using the Edit tool, then re-read the modified files to confirm the replacements look correct.

## Step 7: Summary

Print a final summary:

```
MX2 design token audit complete

Files scanned: X
Files modified: Y
Violations fixed: Z
  - Exact token matches: A
  - Approximate token matches: B
  - No token available (skipped): C
MUI color imports removed: D
Non-MX2 theme.palette usages fixed: E
```
