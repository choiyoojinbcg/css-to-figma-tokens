# Figma Variables JSON Format Specification

This is the exact format Figma expects when importing variables via the Variables Import feature.

## Top-Level Structure

```json
{
  "<category-1>": { /* tokens */ },
  "<category-2>": { /* tokens */ },
  ...
  "$extensions": {
    "com.figma.modeName": "<ModeName>"
  }
}
```

Categories are arbitrary grouping keys (e.g. `spacing`, `color`, `base`, `alpha`). Within a category, tokens can be flat (`spacing.4`) or nested (`color.blue.500`).

## Token Shape

Every leaf token has this structure:

```json
{
  "$type": "<type>",
  "$value": <value>,
  "$extensions": {
    "com.figma.variableId": "VariableID:X:Y",
    "com.figma.scopes": ["<SCOPE>"],
    "com.figma.aliasData": { /* L2 only */ }
  }
}
```

### `$type` values
- `"color"` — for any color token
- `"number"` — for spacing, width, height, radius, font-size, font-weight, line-height, letter-spacing, etc.
- `"fontFamily"` — for font family tokens (e.g. `--font-family-sans`). Value is a plain string with **just the primary font name** (e.g. `"Open Sans"`), not the full CSS fallback stack.
- `"string"` — for arbitrary string tokens (font-style names like `"Regular"` / `"Semi Bold"`, UI copy strings, etc.)
- `"boolean"` — rarely used in CSS, but supported.

### `$value` shape by type

**Number:**
```json
"$value": 16
```
Raw number. Always in pixels for spacing/sizing.

**Color:**
```json
"$value": {
  "colorSpace": "srgb",
  "components": [0.2313725491, 0.5098039216, 0.9647058824],
  "alpha": 1,
  "hex": "#3B82F6"
}
```
- `colorSpace` is always `"srgb"`.
- `components` is `[R, G, B]` with each channel as a float `0.0`–`1.0` (e.g. `255/255 = 1.0`, `59/255 ≈ 0.2313725491`).
- `alpha` is a float `0.0`–`1.0`.
- `hex` is the **6-digit** hex of the RGB channels. **Alpha is NOT encoded in the hex** — even when `alpha < 1`, hex is just the RGB part.

**Font family:**
```json
"$value": "Open Sans"
```
Plain string. Always extract **just the first font name** from a CSS font stack — strip quotes, ignore the fallbacks. So `'Open Sans', system-ui, sans-serif` → `"Open Sans"`. This matches what Figma actually applies to text layers, and aligns with the Tokens Studio convention that the font family value must exactly match what Figma's design panel shows.

**String:**
```json
"$value": "Semi Bold"
```
Plain string. Used for font-style names, weight names ("Regular", "Bold"), or any other named string token.

## VariableID Generation

Format: `VariableID:<collectionIndex>:<variableIndex>`

- **Collection index**: Usually `1` for a single-mode (L1) file. For L2, each mode shares the same collection index because they're modes of the same collection.
- **Variable index**: Increments per token, starting from `3` (slots 1 and 2 are reserved for the collection metadata itself).

For L2, **the same logical token gets the same VariableID across all mode files**. Walk the token list in a stable order (alphabetical within category, categories in input order) once, assigning IDs. Then write each mode file using those same IDs.

Some observed pattern variants:
- Most variables use the simple `VariableID:1:N` pattern.
- Some categories (like `alpha` or sidebar tokens) use different collection indices (`VariableID:42:355`, `VariableID:43:356`, `VariableID:3278:19778`). These reflect the order in which the variables were *created* in Figma, not their position in the file. **For generated output, the simple incremental pattern within a single collection (`VariableID:1:N`) works fine** — Figma will renumber on import anyway.

## Scope Values

`com.figma.scopes` is an array of strings telling Figma which design contexts the variable applies to.

| Category in input | `$type` | Scopes |
|---|---|---|
| `spacing`, `space`, `gap`, `padding`, `margin` | `number` | `["GAP"]` |
| `width`, `w-*`, `height`, `h-*`, `size` | `number` | `["WIDTH_HEIGHT"]` |
| `radius`, `rounded`, `border-radius` | `number` | `["CORNER_RADIUS"]` |
| `font-size`, `text-size` | `number` | `["FONT_SIZE"]` |
| `font-weight` | `number` | `["FONT_WEIGHT"]` |
| `line-height`, `leading` | `number` | `["LINE_HEIGHT"]` |
| `letter-spacing`, `tracking` | `number` | `["LETTER_SPACING"]` |
| `stroke-width`, `border-width` | `number` | `["STROKE_FLOAT"]` |
| `font-family`, `typeface` | `fontFamily` | `["FONT_FAMILY"]` |
| `font-style` (named, e.g. "Regular") | `string` | `["FONT_STYLE"]` |
| Any color | `color` | `["ALL_SCOPES"]` |
| Unknown / misc | matches value | `["ALL_SCOPES"]` |

For colors, use `["ALL_SCOPES"]` by default. More specific scopes like `FRAME_FILL`, `SHAPE_FILL`, `TEXT_FILL`, `STROKE_COLOR` are also valid but `ALL_SCOPES` is the safe default and matches the sample outputs.

## `aliasData` (cross-collection references)

`com.figma.aliasData` tells Figma to make this variable an alias of another variable in a different collection. Used in architecture C and D.

```json
"com.figma.aliasData": {
  "targetVariableId": "",
  "targetVariableName": "color/zinc/50",
  "targetVariableSetId": "",
  "targetVariableSetName": "1. Primitives"
}
```

### How Figma resolves aliases at import time

Per Figma's docs, the resolution is a fallback chain:
1. First try to match `targetVariableId` against an existing variable's ID. (Usually empty in our generated files since we don't know real IDs in advance.)
2. If that fails, look for a variable matching `targetVariableName` within a collection with ID matching `targetVariableSetId`. (Also usually empty.)
3. If that fails, look for a variable matching `targetVariableName` within a collection whose name matches `targetVariableSetName`.
4. If even that fails, the variable gets a literal value (the `$value` field) instead of an alias.

This means **the user's Figma collection name must exactly match `targetVariableSetName`** (case-sensitive). Always confirm this with the user before generating aliased files.

### `targetVariableName` uses SLASHES

This is the most common source of broken imports. When Figma imports nested JSON, it flattens the path with forward slashes.

A primitive in this JSON:
```json
"color": {
  "zinc": {
    "50": { "$type": "color", ... }
  }
}
```
…becomes a variable named **`color/zinc/50`** in Figma (not `color.zinc.50`, not `color/zinc-50`).

So when an aliasData references this primitive, it MUST use:
```json
"targetVariableName": "color/zinc/50"
```

If you use hyphens (`color/zinc-50`) or dots (`color.zinc.50`), Figma won't find the target.

**Conversion rule:** to convert a dotted JSON path into a `targetVariableName`, replace every dot (`.`) with a slash (`/`). Don't touch existing hyphens — they're part of the leaf name.

| Primitive JSON path | `targetVariableName` |
|---|---|
| `color.zinc.50` | `color/zinc/50` |
| `color.bg.surface` | `color/bg/surface` |
| `color.bg.border-strong` | `color/bg/border-strong` |
| `color.border.border` (repeated leaf) | `color/border/border` (keep the repeat) |
| `tailwind colors.base.white` | `tailwind colors/base/white` (spaces preserved) |

### Naming conventions for theme/mode aliases

For architecture B's mode files (Light/Dark), **no aliasData is needed** — both files just have the resolved value and the same VariableID. Aliases are only needed when crossing collection boundaries.

For architecture C (semantic → primitives), `targetVariableName` is the primitive's nested path (slash-form).

For architecture D's mode layer (3. Mode) aliasing the theme layer (2. Theme), the target naming convention is `colors/<role>-<mode>`:
- `colors/background-light`, `colors/background-dark`
- `colors/accent-foreground-light`, `colors/accent-foreground-dark`

The `<mode>` suffix is the lowercased mode name, with spaces converted to hyphens (e.g. `"Brand A"` → `brand-a`).

### Where NOT to use aliasData

- **Tokens that are themselves primitives** (architecture A's output): no aliasData. They ARE the source.
- **Architecture B mode files**: no aliasData. Same-collection mode variants don't need cross-collection references.
- **Self-contained ramps** like `alpha.*` in semantic files: no aliasData. They don't reference primitives.

## Mode Metadata

At the root of each file:

```json
"$extensions": {
  "com.figma.modeName": "<Name>"
}
```

- L1: `"Default"`
- L2: `"Light"`, `"Dark"`, `"Brand A"`, etc.

## Key Ordering

Match this order within each token (Figma is tolerant of order, but matching the samples avoids diff noise):
1. `$type`
2. `$value`
3. `$extensions`

Within `$extensions`:
1. `com.figma.variableId`
2. `com.figma.scopes`
3. `com.figma.aliasData` (if present)

Within color `$value`:
1. `colorSpace`
2. `components`
3. `alpha`
4. `hex`
