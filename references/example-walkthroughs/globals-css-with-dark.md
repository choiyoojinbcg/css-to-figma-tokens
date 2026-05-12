# Walkthrough: Basic `globals.css` with Light + Dark

**Input pattern:** A standard shadcn-style `globals.css` with `:root` defining default (Light) tokens and a `.dark` selector overriding the same tokens for Dark mode. No primitive scales declared — only semantic role names.

**Architecture:** B (single collection, multiple modes).

**Output:** Two files — `Light_tokens.json` and `Dark_tokens.json` — that share VariableIDs.

## Example input

```css
:root {
  --background: #FFFFFF;
  --foreground: #18181B;
  --primary: #18181B;
  --primary-foreground: #FAFAFA;
  --border: #E4E4E7;
  /* ...etc */
}

.dark {
  --background: #09090B;
  --foreground: #FAFAFA;
  --primary: #FAFAFA;
  --primary-foreground: #18181B;
  --border: #27272A;
  /* ...etc */
}
```

## Plan to confirm with the user

> Detected two modes: **Light** (from `:root`) and **Dark** (from `.dark`). I'll generate **two files** with matching VariableIDs so Figma treats them as the same variables in two modes.
>
> - 27 semantic colors → `base.*`
> - No primitives in this file, so architecture **B** (no aliasData needed)
>
> Confirm: ok to proceed?

## Generation steps

1. Parse each selector's variables into `(name, hex, mode)` tuples.
2. Sort tokens alphabetically by name to get a stable order.
3. Assign `VariableID:1:N` starting at N=3. The same name gets the same ID across both modes.
4. For each token in each mode, build the standard color token shape (sRGB components from the hex).
5. NO `aliasData` (architecture B doesn't need it).
6. Set `$extensions.com.figma.modeName` per file: `"Light"` and `"Dark"`.

## Sample output token

```json
"base": {
  "background": {
    "$type": "color",
    "$value": {
      "colorSpace": "srgb",
      "components": [1, 1, 1],
      "alpha": 1,
      "hex": "#FFFFFF"
    },
    "$extensions": {
      "com.figma.variableId": "VariableID:1:5",
      "com.figma.scopes": ["ALL_SCOPES"]
    }
  }
}
```

Same `VariableID:1:5` appears in `Dark_tokens.json` for `base.background` (different value).

## Import in Figma

1. Open Variables modal → create new collection.
2. Drag both `Light_tokens.json` and `Dark_tokens.json` in at the same time.
3. Figma creates one collection with two modes, and matches the same variable across both files by ID.

## What to tell the user

- Files generated and where they are.
- Token count.
- Import order: drag both files in **at the same time** (not one then the other — Figma would create separate collections).
- Anything skipped (shadows, gradients, etc.) and why.
