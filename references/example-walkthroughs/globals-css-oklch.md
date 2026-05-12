# Walkthrough: Modern Tailwind v4 `globals.css` (`oklch()` + `@theme inline`)

**Input pattern:** Tailwind v4-style CSS where `:root` and `.dark` use `oklch()` for colors and a separate `@theme inline` block maps `--color-*` Tailwind aliases to the same tokens. Common in fresh shadcn 2025+ scaffolds.

**Architecture:** B by default (single collection, multiple modes). Upgrade to C if the user wants a synthesized primitives layer.

## Example input

```css
@import "tailwindcss";
@custom-variant dark (&:is(.dark *));

:root {
  --background: oklch(0.9789 0.0082 121.6272);
  --foreground: oklch(0 0 0);
  --primary: oklch(0.5106 0.2301 276.9656);
  --radius: 1rem;
  --font-sans: DM Sans, sans-serif;
  --shadow-sm: 0px 0px 0px 0px hsl(0 0% 10.1961% / 0.18), ...;
  --spacing: 0.23rem;
  /* ...etc */
}

.dark {
  --background: oklch(0 0 0);
  --foreground: oklch(1.0000 0 0);
  --primary: oklch(0.6801 0.1583 276.9349);
  /* ...etc */
}

@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  /* ...etc */
  --radius-sm: calc(var(--radius) - 4px);
  --radius-md: calc(var(--radius) - 2px);
}

@layer base {
  /* ... */
}
```

## Key things to handle

### 1. Skip the `@theme inline` block entirely

Every variable in `@theme inline` is just an alias of one in `:root` (`--color-background: var(--background)`). It exists only for Tailwind's class generator. The skill should:
- Detect `@theme inline` (or `@theme`) blocks
- Skip them completely — don't even parse them
- Use `:root` and `.dark` as the canonical source

### 2. Skip `calc()` expressions

Lines like `--radius-sm: calc(var(--radius) - 4px)` can't be represented as Figma Variables. Skip with a warning. Tell the user to recreate as a literal or compute the value themselves.

### 3. Convert `oklch()` to sRGB

Every color in this file uses `oklch(L C H)`. The conversion chain is `oklch → oklab → linear RGB → sRGB`. See `references/color-conversion.md` for the formulas. Out-of-gamut colors must be clipped — if you clip, flag it to the user so they know there may be slight color shift in Figma vs. browser.

### 4. Skip shadows

`--shadow-*` tokens are multi-value box-shadow strings. Skip. Tell the user to recreate as Effect Styles in Figma.

### 5. `--radius` should stay as a number

`--radius: 1rem` → `16` (rem × 16). Scope: `CORNER_RADIUS`.

### 6. `--spacing: 0.23rem` is a Tailwind v4 spacing scale factor

This is Tailwind v4's *base spacing unit*. All spacing utilities (`p-1`, `gap-4`) are multiples of this. It's not really a typical Figma variable — you might:
- Convert it to a number (`0.23rem × 16 = 3.68`), scope `GAP`, and note that downstream multiples need to be computed manually.
- Or skip it and warn that Tailwind v4's relative spacing scale doesn't map cleanly to Figma.

### 7. `--font-sans: DM Sans, sans-serif`

Convert as `$type: "fontFamily"`, extract just `"DM Sans"`.

## Plan to confirm with the user

> Detected Tailwind v4 globals.css with two modes (Light from `:root`, Dark from `.dark`). Going with architecture **B**.
>
> - **27 semantic colors** in `oklch()` format — converting to sRGB approximations. Will flag any clipped to gamut.
> - **3 font families** → `fontFamily` tokens. Extracting just the first font name (e.g. `"DM Sans"`).
> - **1 radius** (`--radius: 1rem`) → `16` in CORNER_RADIUS scope.
> - **1 spacing base** (`--spacing: 0.23rem`) → flagging this; Tailwind v4's relative spacing doesn't map cleanly to Figma. Want me to skip it or include as raw `3.68`?
> - **Skipping**: 8 shadow tokens (recreate as Effect Styles in Figma), `@theme inline` block (CSS aliases), `calc()` expressions in `@theme inline`.
>
> Confirm: proceed with B, or want to synthesize a primitives layer (C)?

## If the user wants architecture C

Synthesize primitives by extracting unique color values across both modes. Strategy options:
- **By hex**: each unique color becomes `color/<hex>` (e.g. `color/F5F5F5`).
- **By Tailwind nearest match**: find the closest color in the standard Tailwind palette.
- **By role + theme**: each color becomes `color/<role>-<theme>` (e.g. `color/background-light`, `color/background-dark`). Pragmatic if user doesn't want to invent palette names.

Then build the semantic file (architecture C's top layer) with aliases to those primitives.
