# Category & Token Type Detection

How to look at a variable name + value and decide:
1. What category does it belong to in the output JSON?
2. What `$type` and `scopes` should it get?

## Detection priorities

Use this order — earlier rules win:

### 1. Value-based detection (most reliable)

If the value is a color literal (hex, rgb, rgba, hsl, hsla, oklch, lab, named color), the token is a `color`. No matter what the variable is called.

If the value is a numeric pixel/rem value (`16px`, `1rem`, `0.5rem`, plain numbers), the token is `number`. The scope depends on the variable name (see below).

If the value is a CSS font stack (e.g. `'Open Sans', system-ui, sans-serif`) or a single quoted/unquoted font name, the token is `fontFamily`. Extract just the first font name, strip quotes.

If the value is a plain string that's not a font stack and not a numeric value (e.g. `"Semi Bold"` as a font style name), the token is `string`.

### 2. Name-based detection (for typed/scoped categorization)

After determining the `$type`, look at the variable name to pick the category and scope.

**Number tokens — name patterns:**

| Name contains | Category | Scopes |
|---|---|---|
| `spacing`, `space`, `gap`, `padding`, `margin`, `inset` | `spacing` | `["GAP"]` |
| `width`, `w-`, `min-w`, `max-w` | `width` | `["WIDTH_HEIGHT"]` |
| `height`, `h-`, `min-h`, `max-h` | `height` | `["WIDTH_HEIGHT"]` |
| `radius`, `rounded`, `border-radius` | `radius` | `["CORNER_RADIUS"]` |
| `font-size`, `text-size`, `text-xs`, `text-sm` (etc.) | `font-size` | `["FONT_SIZE"]` |
| `font-weight` | `font-weight` | `["FONT_WEIGHT"]` |
| `line-height`, `leading` | `line-height` | `["LINE_HEIGHT"]` |
| `letter-spacing`, `tracking` | `letter-spacing` | `["LETTER_SPACING"]` |
| `stroke-width`, `border-width` | `stroke-width` | `["STROKE_FLOAT"]` |
| `opacity` (rare as number) | `opacity` | `["OPACITY"]` |
| Anything else | `misc` | `["ALL_SCOPES"]` |

If a single category has wildly different scopes (rare), prefer the most specific scope per token rather than splitting into multiple categories.

**Font family / string tokens — name patterns:**

| Name contains | `$type` | Category | Scopes |
|---|---|---|---|
| `font-family`, `typeface`, `font-sans`, `font-serif`, `font-mono` | `fontFamily` | `font-family` | `["FONT_FAMILY"]` |
| `font-style` (with a string value like "Regular") | `string` | `font-style` | `["FONT_STYLE"]` |
| Any other string-valued token | `string` | `misc` | `["ALL_SCOPES"]` |

For `fontFamily`, **always** strip the CSS fallback stack and keep only the first font name. So `'Open Sans', system-ui, sans-serif` → `"Open Sans"`. Remove single or double quotes.

**Color tokens — name patterns:**

| Name contains | Category | Notes |
|---|---|---|
| Color family + scale: `slate-50`, `blue-500`, `red-900`, `zinc-100`, etc. | `color.<family>.<step>` | Nest under family. See "Tailwind palette" below. |
| `primary`, `secondary`, `accent`, `destructive`, `muted`, `background`, `foreground`, `border`, `input`, `card`, `popover`, `ring` | `base.<name>` | These are *semantic* tokens; usually L2. |
| `sidebar-*`, `chart-*`, etc. | `base.<name>` (keeps the hyphenated name) | Also semantic. |
| `alpha-*`, `opacity-*` (when the *value* is a color with alpha) | `alpha.<step>` | Self-contained, no aliasData. |
| Other custom colors | `custom colors.<name>` | Fallback. Always include the space in the key. |

All color tokens get scope `["ALL_SCOPES"]` by default.

## Tailwind palette nesting

When you see Tailwind-style scales (e.g. `--color-blue-50`, `--color-blue-100`, ..., `--color-blue-950`), nest them:

```json
"color": {
  "blue": {
    "50": { "$type": "color", ... },
    "100": { "$type": "color", ... },
    ...
  },
  "slate": {
    "50": { ... }
  }
}
```

Detect Tailwind palettes by matching `<family>-<numeric-step>` where step is one of `50, 100, 200, 300, 400, 500, 600, 700, 800, 900, 950`.

Standard Tailwind families to recognize: `slate, gray, zinc, neutral, stone, red, orange, amber, yellow, lime, green, emerald, teal, cyan, sky, blue, indigo, violet, purple, fuchsia, pink, rose`.

## Custom (non-Tailwind) palette nesting

Most real projects mix Tailwind-style palettes with custom brand palettes (e.g. `--color-brand-navy`, `--color-stage-1`, `--color-lane-nbn-bg`). Apply this rule:

**`--color-<group>-<rest>` → `color.<group>.<rest>`**

The `<group>` is the next segment after `color-`, and `<rest>` is everything after that (keep hyphens in the leaf key).

Examples:
- `--color-brand-navy` → `color.brand.navy`
- `--color-brand-blue` → `color.brand.blue`
- `--color-stage-1` → `color.stage["1"]`
- `--color-stage-8` → `color.stage["8"]`
- `--color-lane-nbn-bg` → `color.lane["nbn-bg"]`
- `--color-lane-delivery-text` → `color.lane["delivery-text"]`
- `--color-bg-page` → `color.bg.page`
- `--color-text-primary` → `color.text.primary`

If a color variable has no group (e.g. `--accent-blue` directly, without a `color-` prefix), promote it to a sensible top-level key like `color.accent.blue` based on context.

## Layout / sizing primitive tokens

Custom layout tokens like `--stage-width`, `--header-height`, `--board-padding-x`, `--container-max-width` don't fit cleanly into `spacing`/`width`/`height`. Group them under a `layout` category, and pick the scope per token by looking at the suffix:

- Token name contains `width` → scope `["WIDTH_HEIGHT"]`
- Token name contains `height` → scope `["WIDTH_HEIGHT"]`
- Token name contains `padding`, `margin`, `gap`, `inset` → scope `["GAP"]`
- Anything else with a number value → scope `["ALL_SCOPES"]`

Example:
```json
"layout": {
  "stage-width":      { "$type": "number", "$value": 280, ...scopes: ["WIDTH_HEIGHT"] },
  "header-height":    { "$type": "number", "$value": 48,  ...scopes: ["WIDTH_HEIGHT"] },
  "board-padding-x":  { "$type": "number", "$value": 24,  ...scopes: ["GAP"] }
}
```

## Spacing key conventions

Spacing tokens in the sample use *short* keys: `"0"`, `"4"`, `"px"`, `"0-5"` (representing `0.5`), `"1-5"` (`1.5`), etc.

When converting:
- `--spacing-0` → key `"0"`, value `0`
- `--spacing-4` → key `"4"`, value `16` (assuming 1 unit = 4px, which is Tailwind's default — multiply input number by 4)
- `--spacing-px` → key `"px"`, value `1`
- `--spacing-0\.5` or `--spacing-0_5` → key `"0-5"`, value `2`

If the user's CSS uses *direct* px values rather than Tailwind multiples, just use the px value directly and the key can be the px number as a string.

When uncertain, ask the user whether their spacing scale uses Tailwind's 4px-per-unit convention or direct pixel values.

## Width key conventions

Width tokens use a `w-` prefix in the key:
- `--width-0` or `w-0` → key `"w-0"`, value `0`
- `--width-full` → key `"w-full"`, but only if the value can be represented as a number (skip percentages/keywords).
- `--width-px` → key `"w-px"`, value `1`

Same for height with `h-` prefix.

## Semantic vs. primitive distinction

A token is **semantic** if its name describes its *role* (e.g. `background`, `accent`, `border`, `destructive`) rather than its *value characteristics* (e.g. `blue-500`, `slate-100`).

Semantic tokens belong in `base` (or another role-based group) and, in L2, get `aliasData`.

Primitive tokens belong in their value-based category (`color.blue.500`, `spacing.4`) and never get `aliasData`.

If the input mixes both (common for L2 themes), separate them into the appropriate categories in the output.
