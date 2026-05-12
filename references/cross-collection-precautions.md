# Cross-Collection Reference Precautions

When generating Figma token files that span multiple collections (architectures C and D), several details must be exactly right or the import will fail or partially resolve. This is the single authoritative reference for those details.

**Read this whenever generating aliased files** (any file containing `com.figma.aliasData`).

## TL;DR — The five things that must be right

1. **`targetVariableSetName` matches the Figma collection name exactly** (case-sensitive, spaces and punctuation preserved).
2. **`targetVariableName` uses forward slashes** matching the nested JSON path (`color/zinc/50`, not `color.zinc.50` or `color/zinc-50`).
3. **Collections are imported in dependency order** — primitives first, then anything that aliases them.
4. **Mode files for the same collection are imported together** — drag them in simultaneously, not sequentially.
5. **VariableIDs match across mode files of the same collection** — same logical token gets the same ID in Light and Dark.

If any of these is wrong, Figma silently substitutes literal values for failed aliases (the design still renders correctly using `$value`, but the alias link is missing and changing the primitive won't propagate).

---

## Detail 1: `targetVariableSetName` must match exactly

Figma resolves aliases by walking through existing collections in the file and looking for one whose **name** matches `targetVariableSetName` (case-sensitive). Some examples of mismatches that cause silent failures:

| Generated `targetVariableSetName` | Actual Figma collection name | Result |
|---|---|---|
| `"1. Primitives"` | `"1. Primitives"` | ✅ Resolves |
| `"1. Primitives"` | `"1. primitives"` | ❌ Case mismatch |
| `"1. Primitives"` | `"Primitives"` | ❌ Number prefix missing |
| `"Collection"` | `"Collection 1"` | ❌ Different name |
| `"2. Theme"` | `"2.Theme"` (no space) | ❌ Space matters |

**Always ask the user for the exact name before generating.** Don't guess.

If the user doesn't have the file open in Figma right now, suggest a default name and explicitly tell them: *"After import, make sure your collection in Figma is named exactly `<X>` — case-sensitive. You can rename it in the Variables panel anytime."*

---

## Detail 2: `targetVariableName` must use slashes, not dots or hyphens

Figma normalizes nested JSON keys into forward-slash paths on import. So a primitive defined as:

```json
"color": {
  "zinc": {
    "50": { "$type": "color", "$value": {...} }
  }
}
```

becomes a variable named **`color/zinc/50`** inside Figma. Any aliasData pointing to it must use this exact slashed form:

```json
"com.figma.aliasData": {
  "targetVariableName": "color/zinc/50",   ← slashes
  "targetVariableSetName": "1. Primitives"
}
```

**Conversion rule:** dotted JSON path → slash-separated string. Replace each `.` with `/`. Leave hyphens within a single key alone — they're part of the leaf name.

| Primitive JSON path | `targetVariableName` |
|---|---|
| `color.zinc.50` | `color/zinc/50` |
| `color.bg.surface` | `color/bg/surface` |
| `color.bg.border-strong` | `color/bg/border-strong` (hyphen preserved) |
| `color.border.border` | `color/border/border` (don't collapse the repeat) |
| `tailwind colors.base.white` | `tailwind colors/base/white` (space preserved in top key) |

**Common mistake we've actually hit:** using hyphens between path segments (`color/bg-page` instead of `color/bg/page`). The whole import fails silently this way.

---

## Detail 3: Import order matters

Figma resolves aliases **at the moment of import**, by searching collections that already exist in the file. If the target collection doesn't exist yet, the alias has nothing to resolve to → silent fallback to literal value.

**Required order for architecture C:**
1. Import primitives → new collection (e.g. "1. Primitives")
2. Import semantic → new collection. Aliases resolve against the primitives.

**Required order for architecture D:**
1. Import primitives → new collection (e.g. "1. Primitives")
2. Import theme → new collection. Aliases resolve against primitives.
3. Import mode files (Light + Dark together) → new collection. Aliases resolve against theme.

If a user reports broken aliases after import, **the first question to ask is: "Did you import the primitives collection first, before the semantic/theme files?"**

---

## Detail 4: Multi-mode files must be imported simultaneously

For architecture B (or any case with multiple mode files for the same collection), Figma needs all mode files **at once** to recognize them as modes of the same collection.

- ❌ Wrong: import `Light_tokens.json` (creates a collection with one mode "Light"), then import `Dark_tokens.json` (creates a SEPARATE collection with one mode "Dark").
- ✅ Right: select both files together in the file picker and import in one operation. Figma sees the matching VariableIDs across files and creates one collection with two modes.

If the user accidentally imported them as separate collections, they need to either:
- Delete both collections and reimport simultaneously, OR
- Use Figma's "Import mode" feature on an existing collection to add the second mode.

---

## Detail 5: VariableIDs must match across mode files

This is the technical underpinning of detail 4. For a token to appear as the same variable in two modes, both mode files must use the **same** `com.figma.variableId` for that token. The skill computes IDs once in a stable order (alphabetical within category) and reuses them across all mode files — see step 4 in SKILL.md.

If IDs don't match, Figma sees two separate variables that happen to share a name.

---

## Troubleshooting: when import fails or partially fails

### "Encountered errors importing N tokens"

This usually means N tokens have aliasData that doesn't resolve. Common causes:

1. **`targetVariableSetName` doesn't match the actual Figma collection name.** Open the Variables panel, check the exact collection name (case-sensitive), and either:
   - Rename the collection in Figma to match the JSON, OR
   - Regenerate the JSON with the correct `targetVariableSetName`.

2. **`targetVariableName` uses wrong separators.** Should be slashes throughout. Check that no `targetVariableName` contains a `.` or has hyphens where dots used to be.

3. **The target collection wasn't imported yet.** If you're importing semantic/theme files but haven't imported primitives first, all aliases will fail. Delete the failed import, import primitives first, then re-import.

4. **Variable doesn't exist in the target collection.** If `color/zinc/50` is referenced but the primitives file doesn't actually contain a `color.zinc[50]` token, the alias fails. Verify the primitive exists.

### Aliases imported as literal values (no error, but no link)

When Figma can't resolve an alias, it falls back to the literal value silently. The design renders correctly but changing the primitive won't propagate to the semantic. To detect:
- In the Variables panel, click an aliased variable. If you see a hex value instead of `→ {primitive name}`, the alias didn't link.
- Compare your generated file's aliasData targets against the actual variable names in the target collection.

Recovery: delete the affected variables in the broken collection, re-import the JSON. Or manually rebind in Figma (right-click variable value → "Create alias").

### Modes were created as separate collections

If after importing `Light_tokens.json` and `Dark_tokens.json` you see two collections instead of one with two modes:
- You imported them sequentially. Delete both, then import simultaneously.
- OR: keep the first one as your collection, then on it use "Import mode" (right-click an existing mode column) and select the second file.

---

## What's safe to do AFTER successful import

Once aliases have resolved successfully, Figma stores them as internal references — not by name. This means:

- **Renaming the target collection** after import does NOT break existing aliases. They keep working because Figma is now using internal IDs.
- **Renaming a target variable** also does NOT break aliases that already resolved.
- **Deleting a target variable** WILL break aliases pointing to it (they fall back to literal values).
- **Re-importing the SAME aliased file** when the target collection has been renamed: Figma may not re-resolve correctly because the `targetVariableSetName` in the JSON still points to the old name. Update the JSON to match the new name before re-importing.

The takeaway: name your collections however you like *after* import, but get the names right *for* import.

---

## When to surface these precautions to the user

When generating files that include `aliasData`, **always end the response with a brief precautions checklist**, e.g.:

> **Before importing:**
> - Make sure your primitives collection in Figma is named exactly **`<X>`** (case-sensitive)
> - Import in this order: primitives first, then this file
> - If you have multiple mode files, select all of them together when importing
>
> **If you see "Encountered errors importing N tokens":** see `cross-collection-precautions.md` in this skill for troubleshooting.

Don't bury this in a wall of text — make it the last thing the user sees before they go import.
