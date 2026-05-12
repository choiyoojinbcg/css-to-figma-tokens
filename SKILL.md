---
name: css-to-figma-tokens
description: Convert design token source files (globals.css, SCSS, JS/TS token objects, Tailwind config, or any other token format) into Figma-ready Variables JSON for direct import via Figma's Variables Import feature. Use whenever a user wants to take their code-side design tokens (CSS custom properties, Sass variables, JS token objects, or similar) and produce JSON files Figma can import — whether they have a single primitive set (one file, one mode), a multi-mode/multi-theme setup (e.g. light + dark, brand A + brand B), or a layered primitive + semantic system requiring linked collections. Trigger on phrases like "convert my tokens to Figma", "turn this CSS into Figma variables", "make this Figma-ready", "generate Figma token JSON", "import these into Figma", or whenever a user shares a globals.css / tokens.css / theme file and mentions Figma. Input-agnostic — Claude parses whatever format is given.
---

# CSS to Figma Tokens Converter

Convert any source-side design token file into Figma Variables Import JSON. The skill is **input-agnostic**: Claude reads CSS, SCSS, JS/TS, JSON, Tailwind config, or any other format and extracts tokens by inspecting variable names and values, not by running a fixed parser.

## Workflow

1. **Read the input file(s)** the user provides.
2. **Pick an architecture** based on the input shape (see [Choosing an Architecture](#choosing-an-architecture)).
3. **Confirm the plan and target collection names with the user** before generating output.
4. **Generate the output JSON(s)** following the format described below.
5. **Save outputs** to `/mnt/user-data/outputs/`, present the files, and **print a skip summary** for anything that couldn't be converted (shadows, gradients, etc.).

## Choosing an Architecture

Figma variables use **collections**. A collection is a group of related variables that share modes. The right architecture depends on what's in the user's source:

| Architecture | When to use | Output |
|---|---|---|
| **A. Single collection, one mode** | Source has one flat set of literal values (`:root` only, no theme overrides). Tailwind palette files, primitive token files. | **1 file**, `modeName: "Default"`. |
| **B. Single collection, multiple modes** | Source has the same semantic variable names redefined per theme (`:root` + `.dark`, light/dark in one file). No explicit primitives layer. | **1 file per mode** with the same VariableIDs across files; `modeName` per file (`"Light"`, `"Dark"`). |
| **C. Two collections — primitives + semantic** | Source has separate primitive scales (e.g. `--blue-500`) AND semantic role tokens (`--background`, `--primary`). OR the user wants to add a semantic layer on top of a primitives file they already have. | **2 sets of files**: primitives (no aliases) + semantic file(s) with `aliasData` referencing the primitives collection. |
| **D. Three collections — primitives + theme + mode** | Full shadcn-style architecture. The user wants both: (1) per-theme variants visible as separate variables (`background-light`, `background-dark`), AND (2) a mode-switching layer (`base/background` with Light/Dark modes). | **Multiple files** spanning three collections. Most advanced — read `references/three-collection-architecture.md` before generating. |

**Default behavior:** Pick the *simplest* architecture that fits the input. If `:root` + `.dark` is the only structure, that's **B**, not C or D — don't over-engineer. Offer to upgrade to C or D only if the user asks or the input explicitly contains primitives.

**Ambiguous cases:** When the input has semantic-only tokens (`--background`, `--primary`) but no primitives, **ask the user**: "Do you want a simple two-mode collection (B), or should I synthesize a primitives layer underneath (C)?"

## Pre-flight: Confirm Collection Names

For any architecture that uses `aliasData` (C and D), the `targetVariableSetName` must match the name of the **collection in the user's Figma file** that holds the target variables. Figma cannot resolve aliases if the names don't match exactly.

**Before generating any aliased file, ask the user:**

> "What's the name of the collection in your Figma file that holds the primitives? (Common answers: `Collection`, `1. Primitives`, `1. TailwindCSS`, or whatever you've renamed it to. Case-sensitive.)"

If the user doesn't know, suggest a default and remind them they can rename the collection in Figma later — `targetVariableSetName` just needs to match whatever they end up calling it.

The same applies to the theme layer in architecture D — confirm both `1. Primitives` and `2. Theme` names (or whatever the user uses).

## Format Reference

For the JSON shape (token structure, sRGB color value, scopes, mode metadata, aliasData), read `references/format-spec.md`. This is the canonical source — don't reproduce its details here.

For deciding what category and `$type` a variable belongs to (color vs number vs fontFamily, what scopes to apply), read `references/category-detection.md`.

For converting color formats (hex, rgb, hsl, **oklch**, lab) into the sRGB component arrays Figma expects, read `references/color-conversion.md`.

For the three-collection (primitives + theme + mode) architecture in detail — including how aliases chain across collections and how to assign IDs across multiple files — read `references/three-collection-architecture.md`.

## Conversion Steps (high level)

1. **Parse the input.** Build `(name, value, mode)` tuples. Strip `--` prefix. Map CSS selectors to modes: `:root` → default/Light, `.dark` → Dark, `.light` → Light, `[data-theme="x"]` → x. Ignore media-query `:root` overrides (those are responsive, not theme modes).

2. **Group by category.** Use the name/value patterns in `references/category-detection.md`. Tailwind palette scales nest under family (`color.blue.500`). Semantic tokens go under `base.<name>`. Layout primitives go under `layout`. Alpha ramps go under `alpha`.

3. **Convert each value.** Numbers: strip `px`, multiply `rem` by 16. Colors: convert to sRGB components (see `references/color-conversion.md` for oklch/hsl/lab). Font families: extract just the first font name.

4. **Assign VariableIDs.** Single collection: `VariableID:1:N` from N=3 upward. Multiple modes of the same collection: **compute IDs once** in a stable order (alphabetical within category, categories in input order), then reuse the same IDs across all mode files. Multiple collections (C/D): each collection gets its own incrementing collection-index (`VariableID:1:N`, `VariableID:2:N`, ...).

5. **Build `$extensions`.** Every token gets `variableId` + `scopes`. Aliased tokens (C/D) also get `aliasData` with `targetVariableName` (slash-separated path matching the target's nested name) and `targetVariableSetName` (the collection name confirmed with the user).

6. **Add mode metadata at the root** of each file: `$extensions.com.figma.modeName: "<Name>"`.

7. **Write outputs and present them.** Filenames should reflect the mode or layer (`Default_tokens.json`, `Light_tokens.json`, `Primitives.json`, `Semantic.json`).

## Edge Cases

Things to handle gracefully — always tell the user what got skipped and why:

- **Variables referencing other variables** (`var(--foo)`): resolve to the literal value inline.
- **`calc()` expressions**: skip and warn — Figma Variables can't store computed expressions.
- **Box-shadow tokens** (multi-value strings): skip and warn — Figma Variables are atomic. The user must recreate these as **Effect Styles** in Figma by hand.
- **Gradients** (`linear-gradient(...)`): skip and warn — recreate as Fill Styles in Figma.
- **Percentages and viewport units** (`100%`, `100vh`): skip and warn — Figma Variables are absolute.
- **Keyword values** (`auto`, `fit-content`, `currentColor`): skip. Exception: `transparent` → `#000000` with alpha `0`.
- **`oklch()` / `hsl()` / `lab()` / `lch()` colors**: convert to sRGB (see `references/color-conversion.md`). Note clipping if out-of-gamut.
- **Font family stacks**: extract just the first font name, strip quotes. `'Open Sans', system-ui, sans-serif` → `"Open Sans"`.
- **`@theme inline` blocks (Tailwind v4)**: these are CSS aliases mapping `--color-X` → `var(--X)`. They duplicate the `:root` tokens for Tailwind's class-name generation. **Skip the `@theme inline` block entirely** and use the `:root` definitions as the source of truth.
- **Unknown categories**: bucket as `misc` with scope `ALL_SCOPES` rather than dropping.

## Sample Outputs

See `references/sample-outputs/` for canonical examples:
- `level-1-default.json` — Architecture A (one file, one mode)
- `level-2-light.json` / `level-2-dark.json` — Architecture B (one file per mode)

Read `references/example-walkthroughs/` for full end-to-end walkthroughs of specific common inputs:
- `globals-css-with-dark.md` — basic shadcn light/dark file → Architecture B
- `globals-css-oklch.md` — modern Tailwind v4 with `oklch()` colors and `@theme inline` → Architecture B or C
- `primitives-plus-semantic.md` — building a semantic layer on top of an existing primitives file → Architecture C
- `three-collection-shadcn.md` — full primitives + theme + mode setup → Architecture D
