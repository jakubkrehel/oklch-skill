---
name: oklch-skill
description: OKLCH color space for web projects. Convert hex/rgb/hsl to oklch, generate palettes, check contrast, handle gamut boundaries, and theme with Tailwind v4. Triggers on oklch, color conversion, palette generation, contrast ratio, gamut, display p3, design tokens, hue drift, chroma, dark mode colors.
---

# OKLCH Colors

## About

OKLCH is a perceptually uniform color space where the numbers actually mean what you think they mean. Equal lightness values look equally bright regardless of hue. Adjusting chroma doesn't shift the hue. Building a palette is arithmetic, not guesswork. Most color problems in CSS — broken palettes, failing contrast, hue drift — come from using color spaces that don't match how we see. OKLCH fixes the model so the tools work.

This skill helps you convert existing colors to oklch, build palettes that hold up across hues and themes, check contrast properly, and understand gamut boundaries so colors don't silently clip. To explore interactively, visit [oklch.fyi](https://oklch.fyi).

## Why OKLCH

**Perceptual uniformity.** Equal steps in L produce equal perceived brightness changes. `oklch(0.5 ...)` is visually halfway between black and white. HSL's `lightness: 50%` is not — it varies wildly by hue.

**Stable hue.** HSL blue (`hsl(240, ...)`) shifts toward purple as lightness changes. OKLCH hue stays constant — `oklch(0.3 0.15 260)` and `oklch(0.8 0.15 260)` are both the same blue.

**Independent chroma.** In HSL, `saturation: 100%` at `lightness: 90%` looks washed out. In OKLCH, chroma is an absolute measure of colorfulness that doesn't depend on lightness.

**Finite gamut.** Not every oklch value maps to a displayable sRGB color. High-chroma values at certain hues will clip. This skill treats gamut awareness as a first-class concern.

## OKLCH Syntax

```
oklch(L C H)
oklch(L C H / alpha)
```

| Channel | Range | Description |
| --- | --- | --- |
| L (Lightness) | 0–1 | 0 = black, 1 = white. Perceptually uniform. |
| C (Chroma) | 0–~0.4 | Colorfulness. 0 = gray. Max depends on L and H. |
| H (Hue) | 0–360 | Hue angle in degrees. |
| alpha | 0–1 | Optional transparency. Slash syntax. |

**Formatting precision:** L and C use 3 decimal places, H uses up to 3. Drop trailing zeros. Format `-0` as `0`.

```css
oklch(0.637 0.237 25.331)
oklch(0.8 0.05 200 / 0.5)
```

Browser support: Baseline 2023, 96%+ global coverage. Safe for production without polyfills.

## Color Conversion

When converting existing colors to oklch, convert the color values but leave everything else unchanged — don't change gradient interpolation, don't restructure the CSS.

### Supported input formats

| Format | Examples |
| --- | --- |
| Hex (3/6/8-digit) | `#f00`, `#ff0000`, `#ff000080` |
| `rgb()` / `rgba()` | `rgb(255, 0, 0)`, `rgba(255, 0, 0, 0.5)` |
| `hsl()` / `hsla()` | `hsl(0, 100%, 50%)`, `hsla(0, 100%, 50%, 0.5)` |

### Conversion examples

```css
/* Before */
color: #3b82f6;
background: #1e293b;
border-color: #e2e8f0;

/* After */
color: oklch(0.623 0.188 259.815);
background: oklch(0.279 0.037 260.031);
border-color: oklch(0.929 0.013 255.508);
```

```css
/* Before */
color: rgb(59, 130, 246);
border: 1px solid rgba(0, 0, 0, 0.1);

/* After */
color: oklch(0.623 0.188 259.815);
border: 1px solid oklch(0 0 0 / 0.1);
```

Alpha uses the forward-slash syntax. Omit alpha when it's 1.

### What to leave alone

- CSS keywords: `currentColor`, `inherit`, `initial`, `unset`, `transparent`
- Gradient interpolation methods — only convert the color stops, not the function itself
- Colors in third-party library configs that expect hex input

### Bulk conversion

When converting an entire file:

1. Replace all hex colors with their oklch equivalents
2. Replace all `rgb()`, `rgba()`, `hsl()`, `hsla()` function calls
3. Leave gradient functions unchanged — only convert the color stops within them
4. Leave `currentColor`, `inherit`, `transparent`, and CSS keywords as-is
5. Preserve comments and formatting

## Palette Generation

### The scale convention

Design system palettes use a numeric scale from 50 (lightest) to 950 (darkest). The standard labels by palette size:

| Size | Labels |
| --- | --- |
| 5 | 100, 300, 500, 700, 900 |
| 9 | 50, 100, 200, 300, 500, 700, 800, 900, 950 |
| 11 | 50, 100, 200, 300, 400, 500, 600, 700, 800, 900, 950 |

9 steps is the standard default (matches Tailwind).

### Algorithm

Given a base color with lightness (L), chroma percentage, and hue (H):

**Step 1 — Lightness bounds:**

```
delta = 0.4
minL = max(0.05, baseL - delta)
maxL = min(0.95, baseL + delta)
```

Lightness is clamped to [0.05, 0.95] to avoid pure black/white which have zero chroma.

**Step 2 — Distribute lightness** evenly from maxL (lightest, label 50) to minL (darkest, label 950).

**Step 3 — Clamp chroma per step.** Each lightness level has a different maximum chroma for a given hue and color space:

```
maxChroma = findMaxChroma(step[i].L, hue, colorSpace)
step[i].C = (chromaPercentage / 100) * maxChroma
```

This ensures every step is within gamut. High-chroma base colors will have lower chroma at the lightest and darkest ends — this is correct and expected.

### CSS variable output

```css
:root {
  --color-50: oklch(0.971 0.012 250);
  --color-100: oklch(0.932 0.028 250);
  --color-200: oklch(0.882 0.048 250);
  --color-300: oklch(0.812 0.078 250);
  --color-500: oklch(0.623 0.188 250);
  --color-700: oklch(0.445 0.138 250);
  --color-800: oklch(0.362 0.108 250);
  --color-900: oklch(0.289 0.078 250);
  --color-950: oklch(0.215 0.048 250);
}
```

### Multi-hue palettes

When generating palettes for multiple hues, use the same **lightness** and **chroma percentage** for all. Same L guarantees equal perceived brightness. Same chroma percentage (not absolute chroma) guarantees equal vividness relative to each hue's maximum.

```css
:root {
  /* Same L, same C% (80% of max) — different absolute C per hue */
  --blue-500: oklch(0.623 0.141 250);   /* 80% of max 0.176 */
  --green-500: oklch(0.623 0.157 145);  /* 80% of max 0.196 */
  --red-500: oklch(0.623 0.202 25);     /* 80% of max 0.253 */
}
```

Different hues have different max chroma at the same lightness. Using the same absolute C value across hues would make some appear more vivid than others.

### Dark mode

Reverse the palette mapping so that the lightest step becomes the darkest and vice versa:

```css
:root {
  --color-bg: var(--color-50);
  --color-text: var(--color-950);
}

.dark {
  --color-bg: var(--color-950);
  --color-text: var(--color-50);
}
```

This works because oklch's perceptual uniformity means equal L steps in both directions produce equally readable results.

### Why not HSL palettes?

**Hue drift:** `hsl(240, 80%, 20%)` and `hsl(240, 80%, 90%)` are not the same perceptual hue. The light variant shifts ~18° toward purple. OKLCH hue is stable.

**Brightness inconsistency:** `hsl(60, 100%, 50%)` (yellow) and `hsl(240, 100%, 50%)` (blue) have the same HSL lightness but wildly different perceived brightness.

## Accessibility & Contrast

### WCAG 2 thresholds

| Content Type | AA | AAA |
| --- | --- | --- |
| Normal text (<18px / <14px bold) | 4.5:1 | 7:1 |
| Large text (>=18px / >=14px bold) | 3:1 | 4.5:1 |
| UI components & graphical objects | 3:1 | — |

### APCA thresholds (WCAG 3 draft)

Conservative approximations using Lc (Lightness Contrast):

| Content Type | Pass | Pass+ |
| --- | --- | --- |
| Normal text | Lc 60 | Lc 75 |
| Large text | Lc 45 | Lc 60 |
| UI components | Lc 30 | — |

APCA's Lc value is signed — positive means light text on dark background, negative means dark text on light. Use the absolute value for threshold comparison.

### Why oklch makes contrast fixes easier

In hex/rgb, fixing contrast means trial and error. In oklch, contrast is controlled by **lightness (L) alone**:

```css
/* Failing: text on background */
color: oklch(0.65 0.2 250);
background: oklch(0.75 0.05 250);

/* Fix: increase L difference, keep C and H */
color: oklch(0.35 0.2 250);
background: oklch(0.75 0.05 250);
```

The rule: move the text's L further from the background's L. Chroma has negligible effect on contrast ratios — adjust L instead.

### Quick lightness gap guide

- **Light background (L > 0.85):** text L should be below 0.45
- **Dark background (L < 0.25):** text L should be above 0.75

These are approximations — always verify with an actual contrast calculation.

### Light vs dark color detection

A color is considered light when its oklch lightness exceeds 0.6:

```
if L > 0.6 → use dark text on this background
if L <= 0.6 → use light text on this background
```

### Hue drift detection

To detect hue drift in an existing HSL palette:

1. Convert each step to oklch
2. Compare the H values across steps
3. If the hue spread is greater than 10°, the palette has visible drift

```css
/* HSL blue ramp — hue shifts toward purple */
hsl(240, 80%, 20%)  →  oklch H ≈ 269
hsl(240, 80%, 50%)  →  oklch H ≈ 267
hsl(240, 80%, 90%)  →  oklch H ≈ 285  /* shifted 18° */
```

## Gamut Awareness

### sRGB vs Display P3

Every sRGB color exists within Display P3, but not every P3 color exists within sRGB. Display P3 covers ~50% more colors.

### Max chroma varies by lightness and hue

The gamut boundary is irregular. At L=0.5 in sRGB:

- Highest chroma: purple (H ≈ 285) at C ≈ 0.29
- Red-orange (H ≈ 0-30): C ≈ 0.20
- Lowest chroma: cyan (H ≈ 195) at C ≈ 0.09

The peak hue shifts with lightness. At L=0.7 magenta peaks, at L=0.9 green peaks. Cyan consistently has the lowest max chroma.

### Gamut checking

If a color's chroma exceeds the maximum for its L/H/space, it clips. The fix: reduce chroma while keeping L and H constant.

```css
/* Out of sRGB gamut */
oklch(0.7 0.35 150)

/* Clamped to max chroma */
oklch(0.7 0.22 150)
```

### CSS fallback patterns

```css
/* sRGB fallback for all browsers */
.accent {
  color: oklch(0.7 0.2 150);
}

/* P3 enhancement for wider gamut displays */
@media (color-gamut: p3) {
  .accent {
    color: oklch(0.7 0.3 150);
  }
}
```

For browsers without oklch support:

```css
.accent {
  color: #4ade80;
}

@supports (color: oklch(0 0 0)) {
  .accent {
    color: oklch(0.7 0.2 150);
  }

  @media (color-gamut: p3) {
    .accent {
      color: oklch(0.7 0.3 150);
    }
  }
}
```

### P3 benefit by hue (measured at L=0.6)

| Hue | sRGB Max C | P3 Max C | Gain |
| --- | --- | --- | --- |
| Greens (H ≈ 145) | 0.189 | 0.256 | +35% |
| Cyans (H ≈ 195) | 0.102 | 0.138 | +35% |
| Reds (H ≈ 25) | 0.243 | 0.274 | +13% |
| Magentas (H ≈ 330) | 0.273 | 0.300 | +10% |

Greens and cyans benefit most from P3. Blues and purples have the smallest gap.

### Don't use P3 when

- The color is a neutral (C < 0.05) — no visible difference
- You can't provide an sRGB fallback
- The design system requires exact hex matching with external tools

## Tailwind v4

Tailwind CSS v4 defines its default palette in oklch. Custom themes should follow the same convention.

### Custom color scale with @theme

```css
@theme {
  --color-brand-50: oklch(0.971 0.012 250);
  --color-brand-100: oklch(0.932 0.028 250);
  --color-brand-200: oklch(0.882 0.048 250);
  --color-brand-300: oklch(0.812 0.078 250);
  --color-brand-400: oklch(0.722 0.148 250);
  --color-brand-500: oklch(0.623 0.188 250);
  --color-brand-600: oklch(0.535 0.168 250);
  --color-brand-700: oklch(0.445 0.138 250);
  --color-brand-800: oklch(0.362 0.108 250);
  --color-brand-900: oklch(0.289 0.078 250);
  --color-brand-950: oklch(0.215 0.048 250);
}
```

This gives you `bg-brand-500`, `text-brand-200`, etc. automatically.

### Opacity modifiers

Tailwind's opacity modifier syntax works with oklch:

```html
<div class="bg-brand-500/50"></div>
<!-- Compiles to: oklch(0.623 0.188 250 / 0.5) -->
```

### Migrating existing themes

1. Convert all hex values in `@theme` to oklch
2. Replace any `theme()` references that used hex
3. Test dark mode — oklch values may look slightly different due to perceptual accuracy
4. Check for hardcoded hex in component code and convert those too

## Review Checklist

| Issue | Fix |
| --- | --- |
| Hex/rgb/hsl color in new code | Convert to `oklch()` |
| HSL palette ramp with hue drift | Rebuild with constant oklch hue |
| Failing contrast ratio | Adjust oklch L channel, keep C and H |
| High chroma without gamut check | Clamp to max chroma for the L/H in sRGB |
| Same absolute C across different hues | Use same C% (percentage of max) for consistent vividness |
| P3 color without sRGB fallback | Add `@media (color-gamut: p3)` pattern |
| Dark mode with hand-picked colors | Derive from light palette by reversing L mapping |
| Hex in Tailwind v4 `@theme` | Convert to oklch values |
| Alpha with comma syntax | Use slash: `oklch(L C H / alpha)` |
