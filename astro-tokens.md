Audit the git diff for hardcoded color and spacing styles that should use design system tokens instead. This command is exclusively for the astro-code project.

## Step 0: Parse arguments

Check if the user passed any arguments:

- **`--base <branch>`**: Use `<branch>` as the base for the diff instead of the auto-detected default branch. The next argument after `--base` is the branch name.

**Determine the base branch:**
- If `--base <branch>` was provided, use that value directly.
- Otherwise, auto-detect the default branch by running `git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@'` (if this fails, use `dev`).

Use the resulting base branch for all diff commands below.

## Step 1: Identify changed frontend files

**You MUST run the git diff commands fresh right now** — do NOT reuse any previously observed or cached list of changed files from earlier in this conversation.

Run all three commands and union the results:
- `git diff <base>...HEAD --name-only` (committed changes)
- `git diff --name-only` (unstaged changes)
- `git diff --cached --name-only` (staged changes)

From the combined list, keep only files with these extensions: `.tsx`, `.jsx`, `.ts`, `.js`, `.css`, `.scss`.

**Exclude** these files — they ARE the token definitions and must not be linted:
- Any file named `global.tokens.css`
- Any file named `manual.tokens.css`

If no frontend files were changed, report "No frontend files in diff" and stop.

## Step 2: Build the token lookup

Read both design system token files:
- `AstroSite/astro-site/src/app/global.tokens.css`
- `AstroSite/astro-site/src/app/manual.tokens.css`

Parse every CSS custom property declaration (`--name: value;`) into a lookup map. Resolve `var()` references so you know the final computed value (px or hex) for each token. For example:
- `--gap-3: var(--size-general-4)` and `--size-general-4: 16px` → `--gap-3` resolves to `16px`
- `--color-foreground-primary: var(--color-white)` and `--color-white: #ffffff` → resolves to `#ffffff`

Group tokens by category for lookup priority:
- **Colors**: all `--color-*` tokens
- **Gaps**: `--gap-*` tokens
- **Padding**: `--padding-*` tokens
- **Radius**: `--radius-*` tokens
- **Border widths**: `--border-*` tokens (where values are thickness, not colors)
- **Sizes**: `--size-general-*` tokens (fallback for spacing)

## Step 3: Scan for violations

For each file from Step 1, read it and scan for hardcoded styles that should use tokens. There are two detection modes:

### Mode A: Tailwind utility classes

Search inside `className`, `class`, and `cn()` / `clsx()` / `twMerge()` strings for these patterns:

**Spacing utilities** — flag when the value is a numeric Tailwind spacing value:
`h-`, `w-`, `min-h-`, `max-h-`, `min-w-`, `max-w-`, `p-`, `px-`, `py-`, `pl-`, `pr-`, `pt-`, `pb-`, `m-`, `mx-`, `my-`, `ml-`, `mr-`, `mt-`, `mb-`, `gap-`, `gap-x-`, `gap-y-`, `top-`, `right-`, `bottom-`, `left-`, `inset-`, `inset-x-`, `inset-y-`, `space-x-`, `space-y-`

**Color utilities** — flag when followed by a Tailwind color name/shade:
`bg-`, `text-`, `border-`, `ring-`, `outline-`, `divide-`, `accent-`, `fill-`, `stroke-`, `shadow-`, `placeholder-`, `from-`, `via-`, `to-`, `decoration-`

**Border radius** — flag these specific classes:
`rounded`, `rounded-sm`, `rounded-md`, `rounded-lg`, `rounded-xl`, `rounded-2xl`, `rounded-3xl`, and their directional variants (`rounded-t-`, `rounded-b-`, `rounded-l-`, `rounded-r-`, `rounded-tl-`, etc.)

**Border width** — flag: `border` (1px), `border-2`, `border-4`, `border-8`

### Mode B: Raw CSS values

In `.css`/`.scss` files and inline `style={}` objects in TSX/JSX, flag:

**Hardcoded pixel values** in these properties:
`padding`, `padding-top`, `padding-right`, `padding-bottom`, `padding-left`, `margin`, `margin-top`, `margin-right`, `margin-bottom`, `margin-left`, `gap`, `row-gap`, `column-gap`, `width`, `height`, `min-width`, `max-width`, `min-height`, `max-height`, `top`, `right`, `bottom`, `left`, `border-radius`, `border-width`

**Hardcoded color values** (hex `#xxx`/`#xxxxxx`, `rgb()`, `rgba()`) in these properties:
`color`, `background`, `background-color`, `border-color`, `fill`, `stroke`, `outline-color`, `box-shadow`, `text-decoration-color`

### Exclusions — do NOT flag any of these:

- **Structural layout values**: `w-full`, `h-full`, `w-screen`, `h-screen`, `h-dvh`, `w-dvw`, `w-fit`, `h-fit`, `w-min`, `w-max`, `h-min`, `h-max`, `w-auto`, `h-auto`
- **Zero values**: any utility ending in `-0` (e.g., `p-0`, `gap-0`, `m-0`, `rounded-none`)
- **Special keywords**: `auto`, `fit`, `min`, `max`, `transparent`, `inherit`, `current`, `currentColor`, `none`
- **Already tokenized values**: anything already using `var(--...)` syntax, including Tailwind arbitrary values like `gap-[var(--gap-4)]` or `bg-[color:var(--color-blue)]`
- **Fraction/percentage values**: `w-1/2`, `w-1/3`, `h-1/4`, etc.
- **`rounded-full`**: this is a valid structural value (maps to `border-radius: 9999px` which is `--radius-5`, but `rounded-full` is idiomatic and acceptable)
- **Negative values**: `-m-2`, `-top-4`, etc. — these have no token equivalent
- **Responsive/state prefixes are fine** — still flag the base utility (e.g., in `md:gap-4`, flag `gap-4`)
- **CSS values already using `var()`**: skip any declaration where the value contains `var(--`

## Step 4: Map violations to token replacements

For each violation found in Step 3, determine the replacement:

### Spacing: convert Tailwind value to px, then find the token

Use the standard Tailwind spacing scale:
- `0.5`→2px, `1`→4px, `1.5`→6px, `2`→8px, `2.5`→10px, `3`→12px, `3.5`→14px, `4`→16px, `5`→20px, `6`→24px, `7`→28px, `8`→32px, `9`→36px, `10`→40px, `11`→44px, `12`→48px, `14`→56px, `16`→64px, `20`→80px, `24`→96px, `28`→112px, `32`→128px, `36`→144px, `40`→160px, `44`→176px, `48`→192px, `52`→208px, `56`→224px, `60`→240px, `64`→256px, `72`→288px, `80`→320px, `96`→384px

**Semantic-first resolution order** — check the semantic token category first, fall back to `--size-general-*`:

| Tailwind prefix | Check first | Fallback |
|---|---|---|
| `gap-*`, `gap-x-*`, `gap-y-*`, `space-x-*`, `space-y-*` | `--gap-*` by px value | `--size-general-*` |
| `p-*`, `px-*`, `py-*`, `pt-*`, `pr-*`, `pb-*`, `pl-*` | `--padding-*` by px value | `--size-general-*` |
| `m-*`, `mx-*`, `my-*`, `mt-*`, `mr-*`, `mb-*`, `ml-*` | `--padding-*` by px value (same scale) | `--size-general-*` |
| `rounded-*` | `--radius-*` by px value | `--size-general-*` |
| `border`, `border-2`, `border-4`, `border-8` | `--border-*` by px value | `--size-general-*` |
| `h-*`, `w-*`, `min-h-*`, `max-h-*`, `min-w-*`, `max-w-*`, `top-*`, `right-*`, `bottom-*`, `left-*`, `inset-*` | `--size-general-*` by px value | — |

**Replacement format for Tailwind classes:**
- Spacing: `gap-4` → `gap-[var(--gap-3)]` (if `--gap-3` = 16px = gap-4's px value)
- Height/width: `h-8` → `h-[var(--size-general-8)]`
- Padding: `p-6` → `p-[var(--padding-4)]` (if `--padding-4` = 24px = p-6's px value)
- Radius: `rounded-lg` → `rounded-[var(--radius-3)]`

### Colors: find nearest DS color token

1. Determine the hex value of the Tailwind color class (e.g., `bg-red-500` → `#ef4444`)
2. Search all `--color-*` tokens for an exact hex match
3. If no exact match, find the nearest token by color distance (compare R, G, B channels)
4. Mark the match quality: "exact" or "approximate (nearest: `--color-X`)"

**Replacement format for Tailwind color classes:**
- `bg-white` → `bg-[color:var(--color-white)]`
- `text-red-500` → `text-[color:var(--color-red)]` (nearest match)
- `border-gray-300` → `border-[color:var(--color-gray-300)]` (exact match)

**Note the `color:` prefix** — Tailwind needs it inside arbitrary values for color utilities to distinguish from spacing.

### Raw CSS replacements

- `padding: 16px` → `padding: var(--padding-3)`
- `gap: 24px` → `gap: var(--gap-4)`
- `color: #ff499b` → `color: var(--color-pink)`
- `width: 32px` → `width: var(--size-general-8)`
- `border-radius: 8px` → `border-radius: var(--radius-2)`

## Step 5: Report violations

Present all violations grouped by file. For each violation, show:

```
📄 path/to/file.tsx

  Line 42: gap-4
    → gap-[var(--gap-3)]  (exact match: --gap-3 = 16px)

  Line 58: bg-red-500
    → bg-[color:var(--color-red)]  (approximate: --color-red = #e7000b, Tailwind red-500 = #ef4444)

  Line 73: padding: 16px
    → padding: var(--padding-3)  (exact match)

  Line 91: color: #2a1f4e
    → no existing token — will create in manual.tokens.css
```

At the end of the report, show a summary count:
- Total violations found
- Exact matches
- Approximate matches
- New tokens needed

Then ask the user: **"Which violations should I fix? (all / specific files / specific items / none)"**

Wait for user response before proceeding.

## Step 6: Apply approved fixes

For each violation the user approved:

1. **If an existing token matches** (exact or approximate): Replace the hardcoded value with the token reference using the Edit tool.

2. **If no existing token matches**: First create a new token in `manual.tokens.css`:
   - Place it in the appropriate section (colors with colors, sizes with sizes)
   - Use an AI-inferred semantic name based on the component name, feature area, and usage context. For example:
     - A dark background in an editor field → `--color-editor-field-bg`
     - A custom width for a game card → `--size-game-card-width`
   - **Do not** use generic names like `--color-custom-1` or `--size-custom-42px`
   - Then use the new token in the replacement

Apply all edits, then re-read the modified files to confirm the replacements look correct.

## Step 7: Summary

Print a final summary:

```
✅ Design token audit complete

Files scanned: X
Files modified: Y
Violations fixed: Z
  - Exact token matches: A
  - Approximate token matches: B
  - New tokens created: C

New tokens added to manual.tokens.css:
  --color-editor-field-bg: #2a1f4e
  --size-game-selector-thumb: 40px
```
