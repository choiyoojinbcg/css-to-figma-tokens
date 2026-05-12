# Walkthrough: Three-Collection shadcn-Style (Architecture D)

**Input pattern:** User wants the full shadcn UI kit setup — primitives + theme + mode — observed in the official shadcn Figma kit. This is the most advanced architecture.

**Architecture:** D (three collections, see `references/three-collection-architecture.md` for the conceptual overview).

## When to recommend this

Only when the user explicitly asks for it, OR when they want all of:
1. A reusable primitives layer
2. Light/Dark mode switching
3. Per-theme variants as individually-addressable variables (so designers can force-light a single component on a dark page)

For simpler needs, architectures B or C are better. Don't push D unless the user asks.

## Pre-flight questions

1. **What's the input?** A globals.css with `:root` + `.dark`? Separate primitives file plus semantic CSS? Just semantic-only CSS that needs synthesized primitives?
2. **Collection names** in Figma: confirm what the user will name each of the three collections. Defaults: `1. Primitives`, `2. Theme`, `3. Mode`.
3. **Token coverage**: which semantic tokens does the user want in the mode layer? Default to all of them, but let them narrow.

## Output files

| File | Collection | Modes | Aliases? |
|---|---|---|---|
| `Primitives.json` | 1. Primitives | Default | No |
| `Theme.json` | 2. Theme | Default | → Primitives |
| `Mode_Light.json` | 3. Mode | Light | → Theme (`*-light` variants) |
| `Mode_Dark.json` | 3. Mode | Dark | → Theme (`*-dark` variants) |

Four files total.

## Generation steps

### Step 1: Primitives.json

If the user has a primitives file already (`Default_tokens.json` from an earlier conversion), reuse it. Otherwise, synthesize one:

- Extract every unique color value from `:root` + `.dark`
- Name each: `color/<role>-<theme>` is the simplest pragmatic choice. So `--background: #FFF` in `:root` becomes a primitive named `color/background-light` with value `#FFF`. `--background: #000` in `.dark` becomes `color/background-dark` with value `#000`.

This produces a primitive for every (semantic-role × theme) pair. It's not as elegant as a Tailwind palette, but it's deterministic and works.

Set `modeName: "Default"`. VariableIDs start at `VariableID:1:3`.

### Step 2: Theme.json

For each semantic role, generate two theme variables: `colors/<role>-light` and `colors/<role>-dark`. Each is a single-mode variable that aliases the corresponding primitive.

```json
"colors": {
  "background-light": {
    "$type": "color",
    "$value": { ...resolved: white... },
    "$extensions": {
      "com.figma.variableId": "VariableID:2:3",
      "com.figma.scopes": ["ALL_SCOPES"],
      "com.figma.aliasData": {
        "targetVariableId": "",
        "targetVariableName": "color/background-light",
        "targetVariableSetId": "",
        "targetVariableSetName": "1. Primitives"
      }
    }
  },
  "background-dark": {
    "$type": "color",
    "$value": { ...resolved: black... },
    "$extensions": {
      "com.figma.variableId": "VariableID:2:4",
      ...
      "com.figma.aliasData": {
        "targetVariableName": "color/background-dark",
        "targetVariableSetName": "1. Primitives",
        ...
      }
    }
  }
}
```

Single mode, `"Default"`. VariableIDs use collection index `2`.

### Step 3: Mode_Light.json + Mode_Dark.json

For each semantic role, generate ONE variable in the mode layer with the same VariableID across both files. The Light file aliases `colors/<role>-light`; the Dark file aliases `colors/<role>-dark`.

```json
// Mode_Light.json
"base": {
  "background": {
    "$type": "color",
    "$value": { ...resolved: white... },
    "$extensions": {
      "com.figma.variableId": "VariableID:3:3",
      "com.figma.scopes": ["ALL_SCOPES"],
      "com.figma.aliasData": {
        "targetVariableName": "colors/background-light",
        "targetVariableSetName": "2. Theme",
        ...
      }
    }
  }
}

// Mode_Dark.json
"base": {
  "background": {
    "$type": "color",
    "$value": { ...resolved: black... },
    "$extensions": {
      "com.figma.variableId": "VariableID:3:3",   // SAME as Light
      "com.figma.scopes": ["ALL_SCOPES"],
      "com.figma.aliasData": {
        "targetVariableName": "colors/background-dark",
        "targetVariableSetName": "2. Theme",
        ...
      }
    }
  }
}
```

Modes: `"Light"` and `"Dark"`. VariableIDs use collection index `3`, shared between the two files.

## Import order (CRITICAL)

Figma resolves aliases at import time by looking for the target collection. If the target doesn't exist yet, the alias gets a literal value instead. So:

1. Import `Primitives.json` → new collection (name it as the user specified, e.g. `1. Primitives`)
2. Import `Theme.json` → new collection. Aliases resolve against Primitives.
3. Import `Mode_Light.json` AND `Mode_Dark.json` simultaneously → new collection with two modes. Aliases resolve against Theme.

**If the user imports out of order**, the affected files will have literal-value variables instead of alias references. They'd need to delete those variables and reimport — or manually bind aliases in the Figma UI.

## Tradeoffs to mention

When proposing D, tell the user:
- **More upfront work**: four files, three collections, careful import order.
- **More flexibility downstream**: can apply individual theme variants to specific components, change primitives once and propagate, swap mode globally.
- **Reversibility**: hard. Once they're working in this system, they can't easily collapse back to architecture B without rebinding everything.

If the user just wants light/dark switching and isn't sure they need the layered system, **strongly recommend B first** and offer to upgrade to D later if they outgrow it.
