# Color Value Conversion

The Figma format requires colors as sRGB component arrays (each channel as a float 0.0–1.0) plus a 6-digit hex string. Here's how to convert from each common CSS color format.

## From Hex

`#RRGGBB` or `#RGB`:
- Expand `#RGB` → `#RRGGBB` (e.g. `#abc` → `#aabbcc`).
- Parse each pair as base-16 to get 0–255.
- Divide by 255 → float 0.0–1.0.
- Round to ~10 decimal places of precision (matches Figma's output).
- `alpha` = `1` (no alpha in plain hex).
- `hex` = the uppercase 6-digit hex.

`#RRGGBBAA` (8-digit hex with alpha):
- Same as above for RGB.
- Last pair → alpha (0–255), divide by 255.
- `hex` stays as the 6-digit RGB only (no alpha in the hex field).

Example: `#3B82F6`
```json
{
  "colorSpace": "srgb",
  "components": [0.2313725491, 0.5098039216, 0.9647058824],
  "alpha": 1,
  "hex": "#3B82F6"
}
```

Example: `#09090BCC` (CC = 80% alpha)
```json
{
  "colorSpace": "srgb",
  "components": [0.0352941193, 0.0352941193, 0.0431372561],
  "alpha": 0.8,
  "hex": "#09090B"
}
```

## From rgb() / rgba()

`rgb(59, 130, 246)`:
- Three values → divide each by 255.
- `alpha` = `1`.
- Compute hex by converting each channel back: `'#' + R.toString(16).padStart(2, '0').toUpperCase() + ...`

`rgba(59, 130, 246, 0.5)`:
- Same as above for RGB.
- Fourth value is alpha (already 0–1).

`rgb(59 130 246 / 0.5)` (modern syntax): equivalent to above.

## From hsl() / hsla()

`hsl(217, 91%, 60%)`:
1. Convert HSL to RGB (standard formula):
   ```
   C = (1 - |2L - 1|) × S
   X = C × (1 - |((H / 60) mod 2) - 1|)
   m = L - C/2
   ```
   Then place `(C, X, 0)`, `(X, C, 0)`, `(0, C, X)`, `(0, X, C)`, `(X, 0, C)`, or `(C, 0, X)` depending on the hue sector, add `m` to each, and you have RGB in 0.0–1.0.
2. Use those values directly as `components`.
3. Compute hex by multiplying each by 255, rounding, and converting to hex.

## From oklch() / oklab() / lab() / lch()

These are perceptually uniform color spaces. Conversion to sRGB is non-trivial — there are standard formulas but they involve matrix operations.

**Recommended approach in this skill:** When you see one of these, use a library or Claude's existing knowledge to convert to sRGB hex, then proceed as if the input were hex. The standard conversion chain is:
- `oklch` → `oklab` → linear RGB → sRGB (apply gamma)
- `lch` → `lab` → XYZ → linear RGB → sRGB

If the resulting color is **out of sRGB gamut**, clip each channel to [0, 1] and note this in the user-facing summary so they know there's some color shift.

A Python conversion helper sketch:
```python
import colorsys

def hex_to_components(hex_str):
    hex_str = hex_str.lstrip('#').upper()
    if len(hex_str) == 3:
        hex_str = ''.join(c * 2 for c in hex_str)
    r = int(hex_str[0:2], 16) / 255
    g = int(hex_str[2:4], 16) / 255
    b = int(hex_str[4:6], 16) / 255
    return [r, g, b]

def rgb_to_hex(components):
    r, g, b = [round(c * 255) for c in components]
    return f"#{r:02X}{g:02X}{b:02X}"
```

## From currentColor / transparent / named colors

- `transparent` → `#000000`, alpha 0
- `white` → `#FFFFFF`
- `black` → `#000000`
- Other named colors (`red`, `cornflowerblue`, etc.): look up the CSS named color hex.
- `currentColor`: skip and warn the user — it has no fixed value.

## Precision

Match the precision in the sample outputs (`0.9803921580314636` etc.) — roughly 16 significant digits, which is JavaScript number precision. In Python:

```python
r = int(hex_str[0:2], 16) / 255.0  # gives the right precision
```

Don't manually round to fewer decimals — preserve full float precision.
