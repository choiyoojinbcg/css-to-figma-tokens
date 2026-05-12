# Three-Collection Architecture (Primitives → Theme → Mode)

This is the full shadcn-style design system architecture, observed in production Figma files. It uses three linked collections to give designers maximum flexibility.

## The Pattern

```
┌──────────────────────────────────────────────────┐
│  3. Mode  (collection 3, has Light + Dark modes) │
│                                                  │
│  base/background                                 │
│    ├── Light mode → alias → 2. Theme/colors/     │
│    │                        background-light     │
│    └── Dark mode  → alias → 2. Theme/colors/     │
│                             background-dark      │
└──────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│  2. Theme  (collection 2, single "Default" mode) │
│                                                  │
│  colors/background-light                         │
│    └── Default → alias → 1. Primitives/          │
│                          color/white             │
│                                                  │
│  colors/background-dark                          │
│    └── Default → alias → 1. Primitives/          │
│                          color/zinc/950          │
└──────────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│  1. Primitives  (collection 1, single mode)      │
│                                                  │
│  color/white         = #FFFFFF (literal value)   │
│  color/zinc/950      = #09090B (literal value)   │
│  spacing/4           = 16                        │
│  ...                                             │
└──────────────────────────────────────────────────┘
```

## Why this pattern exists

- **Layer 1 (Primitives)** holds raw color scales, spacing scales, etc. Pure structural values, no semantic meaning. Reusable across themes.
- **Layer 2 (Theme)** holds per-theme variants as separate, individually-addressable variables (`background-light` vs `background-dark`). This gives designers fine-grained control to apply just one theme variant when needed.
- **Layer 3 (Mode)** is the top-level semantic API designers use day-to-day (`base/background`). It has the Light/Dark mode switcher that flips the entire UI.

This pattern means:
- Changing a primitive (e.g. `zinc/950`) automatically updates every theme variant that uses it, which automatically updates the mode layer.
- Designers can apply individual theme variants when needed (e.g. force-light a component on a dark page).
- Mode switching is a one-click operation at the top layer.

## When to use this architecture

Use it (architecture **D** in SKILL.md) only when ALL of these are true:
- The user has separate primitive scales (Tailwind palette, etc.) AND semantic role tokens
- The user wants light/dark (or other) mode switching
- The user wants per-theme variants as separately addressable variables

For simpler cases:
- **`:root` + `.dark` only, no primitives** → Architecture B (single collection, multiple modes)
- **Primitives + semantic roles, no mode switching** → Architecture C (two collections, one mode each)

## Generating the three collections

### Collection 1: Primitives

A single mode (`"Default"`), holds all raw values. Variables have NO `aliasData`.

```
Primitives.json
{
  "color": {
    "zinc": {
      "50":  { "$type": "color", "$value": {...#FAFAFA...}, "$extensions": {...scopes...} },
      "100": { ... },
      ...
      "950": { ... }
    },
    "white": { "$type": "color", "$value": {...#FFFFFF...}, ... }
  },
  "spacing": { ... },
  ...
  "$extensions": { "com.figma.modeName": "Default" }
}
```

VariableIDs use collection index `1`: `VariableID:1:3`, `VariableID:1:4`, etc.

### Collection 2: Theme

A single mode (`"Default"`), but it has **twice the variables** of the mode layer — one for each (token × theme) combination. Each variable aliases a primitive.

```
Theme.json
{
  "colors": {
    "background-light": {
      "$type": "color",
      "$value": { ...resolved value: white... },
      "$extensions": {
        "com.figma.variableId": "VariableID:2:3",
        "com.figma.scopes": ["ALL_SCOPES"],
        "com.figma.aliasData": {
          "targetVariableId": "",
          "targetVariableName": "color/white",
          "targetVariableSetId": "",
          "targetVariableSetName": "1. Primitives"
        }
      }
    },
    "background-dark": {
      "$type": "color",
      "$value": { ...resolved value: zinc/950... },
      "$extensions": {
        "com.figma.variableId": "VariableID:2:4",
        ...
        "com.figma.aliasData": {
          "targetVariableName": "color/zinc/950",
          "targetVariableSetName": "1. Primitives",
          ...
        }
      }
    },
    "foreground-light": { ... },
    "foreground-dark": { ... },
    ...
  },
  "$extensions": { "com.figma.modeName": "Default" }
}
```

VariableIDs use collection index `2`: `VariableID:2:3`, `VariableID:2:4`, etc.

Naming convention for theme variables: `<token-name>-<mode-suffix>` (e.g. `background-light`, `accent-foreground-dark`).

### Collection 3: Mode

**Two files**, one per mode (`Light` and `Dark`), both with the same VariableIDs (shared across the mode column). Each variable aliases the matching theme variable.

```
Mode_Light.json
{
  "base": {
    "background": {
      "$type": "color",
      "$value": { ...resolved value: white... },
      "$extensions": {
        "com.figma.variableId": "VariableID:3:3",
        "com.figma.scopes": ["ALL_SCOPES"],
        "com.figma.aliasData": {
          "targetVariableName": "colors/background-light",
          "targetVariableSetName": "2. Theme",
          ...
        }
      }
    },
    ...
  },
  "$extensions": { "com.figma.modeName": "Light" }
}

Mode_Dark.json
{
  "base": {
    "background": {
      "$type": "color",
      "$value": { ...resolved value: zinc/950... },
      "$extensions": {
        "com.figma.variableId": "VariableID:3:3",   // SAME ID as in Light
        ...
        "com.figma.aliasData": {
          "targetVariableName": "colors/background-dark",
          "targetVariableSetName": "2. Theme",
          ...
        }
      }
    },
    ...
  },
  "$extensions": { "com.figma.modeName": "Dark" }
}
```

VariableIDs use collection index `3`: `VariableID:3:3`, `VariableID:3:4`, etc. **Same ID per token across Light and Dark files** (just like architecture B).

## Import order

Import in this order, top-to-bottom of the dependency chain:

1. **Primitives** first — into a new collection.
2. **Theme** second — into a new collection. Confirm `targetVariableSetName` matches the Primitives collection's actual name in Figma.
3. **Mode (Light)** third — into a new collection. Confirm `targetVariableSetName` matches the Theme collection's name.
4. **Mode (Dark)** fourth — into the same Mode collection as step 3, using "Import mode" rather than creating a new collection.

If the user imports out of order, Figma can't resolve aliases (the target collection doesn't exist yet) and the aliased variables get literal values instead of alias references. They'd then need to either delete and reimport, or rebind aliases manually.

## Synthesizing primitives from semantic-only input

If the user's CSS has only semantic tokens (`--background`, `--primary`) and no primitives, but they want architecture C or D, **synthesize primitives** by extracting unique color values:

1. Collect every unique color value across all modes.
2. Name each primitive. Two strategies:
   - **By hex** (`color/3b82f6`) — deterministic but unfriendly.
   - **By Tailwind nearest match** — find the closest Tailwind palette color (e.g. `color/blue/500`). Most readable, but requires a Tailwind palette reference.
   - **By role + theme suffix** (`color/background-light`) — collapses to the same structure as architecture B's output, just split across two collections. Pragmatic when you don't want to invent palette names.
3. Replace each semantic token's literal value with an alias to the synthesized primitive.

When synthesizing, **always tell the user what naming strategy you used** and offer to redo with a different one.

## Sample chain (from a real Figma file)

For reference, here's a real alias trace observed in a shadcn UI kit:

| Layer | Variable name | Mode | Value or alias target |
|---|---|---|---|
| 3. Mode | `base/background` | Light | → alias `colors/background-light` in `2. Theme` |
| 3. Mode | `base/background` | Dark | → alias `colors/background-dark` in `2. Theme` |
| 2. Theme | `colors/background-light` | Default | → alias `tailwind colors/base/white` in `1. TailwindCSS` |
| 2. Theme | `colors/background-dark` | Default | → alias `tailwind colors/zinc/950` in `1. TailwindCSS` |
| 1. TailwindCSS | `tailwind colors/base/white` | Default | `#FFFFFF` (literal) |
| 1. TailwindCSS | `tailwind colors/zinc/950` | Default | `#09090B` (literal) |

The full primitive name in that file is `tailwind colors/base/white` — note the space and slash. The dotted JSON path `"tailwind colors": { "base": { "white": ... } }` becomes that slashed name on import.
