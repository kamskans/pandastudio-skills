# PandaStudio Skills

Official Claude Code (and multi-agent) skills for [PandaStudio](https://www.writepanda.ai) — the AI desktop video editor for YouTube creators.

## Install in one command

```bash
npx skills add kamskans/pandastudio-skills
```

This installs the `pandastudio` skill into every AI agent you have running — Claude Code, Cursor, Cline, Continue, and [40+ others](https://github.com/vercel-labs/skills#supported-agents).

After install, use `/pandastudio` in Claude Code, or just describe what you want:

> *"Open my latest recording, transcribe it, remove filler words and silences, add bold captions, and export an MP4."*

## What the skill does

The `pandastudio` skill teaches your agent how to drive the PandaStudio desktop app through its localhost CLI + MCP server. It covers:

- **Transcript editing** — transcribe, remove fillers, delete words by ID
- **Motion graphics** — 19 bundled templates + arbitrary HTML/CSS/JS rendering
- **Visual edits** — zooms, speed regions, captions (8 styles), FX, lower thirds, annotations
- **Audio** — DeepFilter noise removal, preview seek
- **AI metadata** — local LLM generates YouTube titles, descriptions, chapter timestamps
- **Export** — MP4 with quality presets, async job polling

## Prerequisites

1. [Download PandaStudio](https://www.writepanda.ai/#download) and install it (macOS or Windows)
2. Enable the automation server: **Settings → Automation → toggle on**
3. Have an AI agent installed (Claude Code, Cursor, Cline, Continue, etc.)

## Tutorials

Step-by-step guides for each agent:

- [Claude Code](https://www.writepanda.ai/tutorials/claude-code)
- [Claude Desktop](https://www.writepanda.ai/tutorials/claude-desktop)
- [Cursor](https://www.writepanda.ai/tutorials/cursor)
- [Continue](https://www.writepanda.ai/tutorials/continue)
- [Cline](https://www.writepanda.ai/tutorials/cline)

## Skill contents

```
skills/pandastudio/
├── SKILL.md                   # Main instructions (frontmatter + editorial rules)
└── reference/
    ├── commands.md            # Every verb.noun with arg schema
    ├── examples.md            # Multi-step recipes
    └── templates.md           # Motion graphic template catalogue
```

## CLI & MCP reference

For the full command surface, see [writepanda.ai/cli](https://www.writepanda.ai/cli).
