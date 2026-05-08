---
name: figma-to-tailwind
description: Reads all Variables from a Figma file URL and writes them as Tailwind CSS v4 @theme tokens into src/index.css.
argument-hint: <figma-url>
allowed-tools: mcp__plugin_figma_figma__use_figma, Read, Edit, Bash
---

# Import Figma Variables as Tailwind v4 Tokens

Fetch all variables from the Figma file URL provided in `$ARGUMENTS` and write them as Tailwind CSS v4 `@theme` custom properties into `src/index.css`.

**Output language:** Respond in the same language the user is using in this conversation.

## Step 1: Parse the URL

Extract the fileKey from `$ARGUMENTS`:

- `figma.com/design/:fileKey/...` → take `:fileKey`
- The `?node-id=` query parameter can be ignored (variables belong to the entire file)

## Step 2: Fetch all variables

Use `use_figma` with the Plugin API to retrieve all variables defined in the file, including unused ones:

```js
const collections = await figma.variables.getLocalVariableCollectionsAsync();
const variables = await figma.variables.getLocalVariablesAsync();

const result = collections.map(col => ({
  id: col.id,
  name: col.name,
  modes: col.modes,
  defaultModeId: col.defaultModeId,
  variables: col.variableIds.map(varId => {
    const v = variables.find(x => x.id === varId);
    if (!v) return null;
    return {
      id: v.id,
      name: v.name,
      type: v.resolvedType,
      valuesByMode: v.valuesByMode
    };
  }).filter(Boolean)
}));

return result;
```

> **Note:** `get_variable_defs` only returns variables already applied to nodes. Always use `use_figma` + Plugin API to retrieve all variables including unused ones.

## Step 3: Map variables to CSS custom properties

Convert Figma variables to Tailwind v4 CSS custom properties.

### Name conversion rules

Convert Figma variable names (slash-separated) to kebab-case:
- Collection name is used as a comment
- The last path segment becomes the property name (e.g. `colors/brand/500` → `brand-500`)
- Slash-separated groups are joined with hyphens (e.g. `font/size/xl` → `text-xl`)
- If Figma uses hyphens to represent decimals in names (`0-5`, `1-5`), convert to underscores in CSS (`0_5`, `1_5`)

### Property name prefix by type

| Figma type | Collection/variable name hint | CSS custom property prefix | Example |
|---|---|---|---|
| `COLOR` | `colors/*` | `--color-` | `--color-brand-500` |
| `FLOAT` | `size/*` | `--spacing-` | `--spacing-4` |
| `FLOAT` | `radius/*` | `--radius-` | `--radius-md` |
| `FLOAT` | `border/*` | `--border-` | `--border-2` |
| `FLOAT` | `font/size/*` | `--text-` | `--text-base` |
| `FLOAT` | `font/weight/*` | `--font-weight-` | `--font-weight-bold` |
| `STRING` | `font/family/*` | `--font-` | `--font-base` |
| `STRING` | other | `--` | `--easing-default` |

### COLOR value conversion

- Figma colors are returned as `{ r, g, b, a }` in the 0–1 range
- Convert `r, g, b` to 0–255 integers and format as two-digit hex: `#rrggbb`
- If alpha < 1.0, use `rgba(r, g, b, a)` format (r, g, b as 0–255 integers)
- Example: `{ r: 0.533, g: 0.414, b: 0.347, a: 1 }` → `Math.round(0.533*255) = 136 = 0x88` → `#886a59`

### FLOAT value conversion

- Convert px to rem (÷ 16)
- Exception: value `0` → `0`; value `1` for px-type variable names → `1px`

### Mode handling

- If a collection has multiple modes (light/dark, etc.), use the value from the **`defaultModeId`** mode

## Step 4: Write to src/index.css

### Check the existing file

Read `src/index.css` and check whether an `@theme` block already exists.

### Write patterns

**Case A: `@theme` block already exists**

Append variables inside the existing `@theme` block. If a key already exists, overwrite (update) it.

**Case B: No `@theme` block exists**

Insert a new `@theme` block immediately after `@import "tailwindcss";`:

```css
@import "tailwindcss";

@theme {
  /* Colors */
  --color-brand-500: #886a59;
}
```

### Output format

Separate sections by collection/group with comments:

```css
@theme {
  /* Brand */
  --color-brand-50: #fff2ea;
  --color-brand-500: #886a59;

  /* Spacing (primitives/size) */
  --spacing-0: 0;
  --spacing-px: 1px;
  --spacing-4: 1rem;

  /* Border Radius (primitives/radius) */
  --radius-none: 0;
  --radius-md: 0.375rem;
  --radius-full: 9999px;

  /* Font Size (primitives/font/size) */
  --text-base: 0.9375rem;
  --text-xl: 1.25rem;

  /* Font Family (primitives/font/family) */
  --font-base: "Noto Sans JP", sans-serif;

  /* Font Weight (primitives/font/weight) */
  --font-weight-bold: 700;
}
```

## Step 5: Report completion

Report the following:
- Number of collections and total variables imported
- List of added/updated CSS custom properties (count per group)
- Mode name used (if multiple modes were present)
- Path to `src/index.css`

## Error handling

- If fileKey cannot be extracted from the URL: ask the user to provide a valid Figma design file URL
- If 0 variables are found: report that the file has no Variables defined
- If `src/index.css` is not found: report that the file does not exist in the current directory and ask the user to confirm the output path
- If `use_figma` returns an error: check the error message before retrying (the operation is atomic — no partial writes occur)
