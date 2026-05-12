# css-to-figma-tokens

A Claude Code skill that converts any design token source file into [Figma Variables Import](https://help.figma.com/hc/en-us/articles/15145852043927) JSON.

Input-agnostic — Claude parses whatever format you give it by inspecting variable names and values, not by running a fixed parser. CSS, SCSS, JS/TS, JSON, Tailwind config, custom DSL, or anything else. Handles single-mode primitives, multi-mode themes (light + dark, brand A + B), and layered primitive + semantic architectures.

## Install

Clone into your Claude Code skills directory:

```bash
git clone https://github.com/choiyoojinbcg/css-to-figma-tokens.git ~/.claude/skills/css-to-figma-tokens
```

That's it — Claude Code auto-discovers the skill on next session. For project-scope install, use `.claude/skills/` instead.

## Usage

Share your token file with Claude Code and ask:

> Convert my tokens to Figma variables
>
> Make this Figma-ready
>
> Turn this into something I can import into Figma

Claude reads the file, picks the right collection architecture, confirms with you, and writes the import-ready JSON.

## What's inside

- `SKILL.md` — the skill definition Claude loads
- `references/` — format spec, color conversion rules, architecture guides, sample outputs, and worked examples
