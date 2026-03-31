# OKLCH Skill

An [Agent Skill](https://docs.anthropic.com/en/docs/claude-code/skills) for working with the OKLCH color space in web projects. Based on [oklch.fyi](https://oklch.fyi).

This skill teaches AI coding assistants (Claude Code, Codex, etc.) to use OKLCH — the perceptually uniform color space that eliminates hue drift, makes palette generation predictable, and simplifies accessibility fixes.

## What it covers

- Converting hex, rgb, hsl to oklch
- Generating perceptually uniform palette scales (50–950)
- Dark mode derivation via lightness manipulation
- WCAG 2 and APCA contrast checking
- Hue drift detection in HSL-based palettes
- sRGB and Display P3 gamut boundaries
- CSS fallback patterns for wider gamut
- Tailwind v4 oklch theming and custom tokens

## Installation

```bash
npx skills add jakubkrehel/oklch-skill
```

## Usage

Once installed, Claude will automatically apply oklch best practices when working with colors — converting formats, generating palettes, checking contrast, and building themes.

You can also invoke it manually:

```
/oklch-skill
```

## License

MIT
