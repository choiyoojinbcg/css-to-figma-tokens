# Walkthrough: Semantic Layer on Existing Primitives (Architecture C)

**Input pattern:** Either (a) one CSS file with both primitive scales (`--blue-500`, `--zinc-100`) AND semantic roles (`--background`, `--primary`), OR (b) a user who already has a primitives file imported in Figma and wants to add a semantic layer that aliases it.

**Architecture:** C (two collections — primitives + semantic).

## Sample mapping

When building the semantic layer, map each semantic role to its source primitive. The mapping depends on context; here's a shadcn-style canonical mapping you can use as a starting point:

| Semantic name | Typical primitive target |
|---|---|
| `background` | `color/zinc/50` or `color/white` |
| `foreground` | `color/zinc/900` |
| `card` | `color/white` |
| `card-foreground` | `color/zinc/900` |
| `popover` | `color/white` |
| `popover-foreground` | `color/zinc/900` |
| `primary` | brand color (e.g. `color/blue/600` or a custom brand primitive) |
| `primary-foreground` | `color/white` |
| `secondary` | `color/zinc/100` |
| `secondary-foreground` | `color/zinc/900` |
| `muted` | `color/zinc/100` |
| `muted-foreground` | `color/zinc/500` |
| `accent` | `color/zinc/100` or brand accent |
| `accent-foreground` | `color/zinc/900` |
| `destructive` | `color/red/500` |
| `destructive-foreground` | `color/white` |
| `border` | `color/zinc/200` |
| `input` | `color/zinc/200` |
| `ring` | `color/zinc/400` or brand color |

Always show the user the mapping you're using and let them adjust before generating.

## Pre-flight questions

1. **What's the primitives collection named in Figma?** (e.g. `Collection`, `1. Primitives`, etc. — case-sensitive)
2. **Confirm the semantic → primitive mapping.** Show the user the table above filled in with their actual primitive names.

## Generation steps

For each semantic token:

1. Look up the value from the target primitive (so the `$value` shows the resolved color — Figma uses this if alias resolution fails).
2. Build the token with `aliasData`:

```json
"base": {
  "background": {
    "$type": "color",
    "$value": { ...resolved value copied from the primitive... },
    "$extensions": {
      "com.figma.variableId": "VariableID:2:3",
      "com.figma.scopes": ["ALL_SCOPES"],
      "com.figma.aliasData": {
        "targetVariableId": "",
        "targetVariableName": "color/zinc/50",
        "targetVariableSetId": "",
        "targetVariableSetName": "<the user's collection name>"
      }
    }
  }
}
```

3. **Critical: `targetVariableName` uses SLASHES**, matching the nested path of the primitive after Figma's import normalization. Dotted JSON keys like `"color": { "zinc": { "50": ... } }` become `color/zinc/50` after import — that's what aliases must reference.
4. VariableID collection index is `2` (different from primitives, which are `1`).
5. Save as `Semantic_tokens.json` (or whatever fits — single mode "Default", or per-mode if doing C with modes).

## Import order

1. Import primitives first (already done by the user, typically).
2. Import the semantic file second. Figma resolves aliases by walking through any collection already in the file and matching `targetVariableSetName` + `targetVariableName`.

If aliases fail to resolve, Figma falls back to the literal `$value` (so nothing breaks visually) but no link is established. The user can then either: rename the primitives collection to match `targetVariableSetName`, OR re-export the semantic file with the correct `targetVariableSetName` and reimport.

## What to tell the user after generating

- Files generated and their location.
- The semantic → primitive mapping you used.
- The exact `targetVariableSetName` value baked into the file.
- Reminder: if their Figma collection has a different name, they need to either rename it OR regenerate.

## Common pitfall: `border-border` style repetition

If the user has a primitive named `color/border/border` (because their CSS had `--color-border-border`), the aliasData should still target the full path `color/border/border`. Don't collapse repetition — Figma's actual variable name preserves the repetition.

## Common pitfall: hyphens vs slashes

When a primitive's name has a hyphen at the leaf (`color/bg/border-strong`), keep the hyphen in the leaf segment. Only the dotted path parts get converted to slashes.

Input: `"color": { "bg": { "border-strong": ... } }` → Figma name: `color/bg/border-strong`
