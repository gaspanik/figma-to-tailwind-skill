# figma-to-tailwind — Figma Variables → Tailwind CSS v4 Skill

A Claude Code skill that reads all Variables defined in a Figma file and writes them as `@theme` tokens into `src/index.css` as Tailwind CSS v4 custom properties — in one command.

---

## What this is

Design tokens locked inside Figma Variables are useless until they reach the codebase. Copying hex values and spacing numbers by hand is error-prone and goes stale the moment the designer makes a change.

This skill automates the entire pipeline: it calls the Figma Plugin API via `use_figma`, fetches every variable in the file (including unused ones), converts them to Tailwind-compatible CSS custom properties, and writes them into `src/index.css` under a single `@theme` block.

```
Figma URL  →  figma-to-tailwind  →  @theme tokens in src/index.css
                                 →  Completion report (collections / variables / properties)
```

---

## What gets converted

| Figma type | Variable name hint | CSS custom property | Example |
|---|---|---|---|
| `COLOR` | `colors/*` | `--color-` | `--color-brand-500: #886a59` |
| `FLOAT` | `size/*` | `--spacing-` | `--spacing-4: 1rem` |
| `FLOAT` | `radius/*` | `--radius-` | `--radius-md: 0.375rem` |
| `FLOAT` | `border/*` | `--border-` | `--border-2: 2px` |
| `FLOAT` | `font/size/*` | `--text-` | `--text-base: 0.9375rem` |
| `FLOAT` | `font/weight/*` | `--font-weight-` | `--font-weight-bold: 700` |
| `STRING` | `font/family/*` | `--font-` | `--font-base: "Noto Sans JP"` |
| `STRING` | other | `--` | `--easing-default: ease-in-out` |

- Slash-separated Figma names become hyphen-separated CSS names (`colors/brand/500` → `--color-brand-500`)
- `FLOAT` values are converted from px to rem (÷ 16), with `0` and `1px` edge cases handled
- `COLOR` values are converted from Figma's 0–1 float format to `#rrggbb` (or `rgba()` when alpha < 1)
- Multi-mode collections: the `defaultModeId` mode is used

---

## Repo structure

```
skills/
  figma-to-tailwind/
    SKILL.md          — skill definition loaded by Claude Code
```

---

## Getting started

**1. Clone this repo**

```bash
git clone https://github.com/gaspanik/figma-to-tailwind-skill
```

**2. Install the skill into Claude Code**

Copy the skill directory into your Claude Code skills folder:

```bash
cp -r skills/figma-to-tailwind ~/.claude/skills/
```

**3. Verify the Figma MCP is connected**

This skill requires the official [Figma MCP server](https://github.com/figma/mcp-server-guide). Confirm it is connected in your Claude Code settings before running the skill.

**4. Run the skill**

Invoke the skill with a slash command or just describe what you want in natural language — both work:

```
/figma-to-tailwind https://www.figma.com/design/<fileKey>/...
```

```
このFigmaファイルの変数をTailwindに書き出して: https://www.figma.com/design/...
```

```
Export Figma variables as Tailwind tokens: https://www.figma.com/design/...
```

The skill writes output directly to `src/index.css` in the current working directory. Run it from your project root.

You will receive a report in this format:

```
✅ Imported 3 collections, 48 variables → 48 CSS custom properties written to src/index.css

  Colors (colors)         — 20 properties
  Spacing (primitives/size) — 14 properties
  Border Radius (primitives/radius) — 6 properties
  Font Size (primitives/font/size)  — 5 properties
  Font Family (primitives/font/family) — 2 properties
  Font Weight (primitives/font/weight) — 1 property

Mode used: light (default)
```

---

## Agent settings

- Use the most capable Claude model available. The skill calls the Figma Plugin API and writes multi-section CSS output — a stronger model handles edge cases (naming collisions, unusual variable structures) more reliably.
- The `SKILL.md` format and `allowed-tools` frontmatter are Claude Code-specific. To use this with another agent (Cursor, Windsurf, etc.), copy the prompt body from `SKILL.md` into that agent's rule format — the conversion logic itself is plain markdown and transfers without modification.

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `fileKey could not be extracted` | Make sure the URL is a Figma design file URL (`figma.com/design/...`). Board and prototype URLs are not supported. |
| `0 variables found` | The file has no Variables defined. Add variables in Figma (Local Variables panel) and re-run. |
| `src/index.css not found` | Run the skill from your project root, or confirm the file exists at `src/index.css`. |
| `use_figma returned an error` | Check that your Figma MCP is connected and has access to the file. Private files require the file owner to grant access. |
| Colors look wrong | Verify the Figma color values are in the 0–1 float range (standard Figma format). Custom plugins that override color storage may produce unexpected output. |

---

## Related

- [tailwind-to-figma-skill](https://github.com/gaspanik/tailwind-to-figma-skill) — the reverse direction: exports Tailwind CSS v4 `@theme` tokens as Figma Variables

---

Built by Masaaki Komori - [@cipher](https://x.com/cipher) · Skill for [Claude Code](https://claude.ai/code) + [Figma MCP](https://github.com/figma/mcp-server-guide)
