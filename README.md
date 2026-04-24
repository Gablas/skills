# skills

Agent skills library. Install via `npx skills` from [skills.sh](https://skills.sh).

## Install

All skills:

```bash
npx skills add Gablas/skills
```

Single skill:

```bash
npx skills add Gablas/skills --skill learn-source
```

Global (all projects):

```bash
npx skills add Gablas/skills -g
```

Specific agent:

```bash
npx skills add Gablas/skills -a claude-code
```

## Skills

| Name | What it do |
|------|------------|
| [`learn-source`](skills/learn-source/SKILL.md) | Research external API / SaaS / webhook source. Output ground-truth `docs/sources/<source>/` bundle. Parallel docs research + live sandbox probes. |
| [`handover`](skills/handover/SKILL.md) | Write self-contained handover prompt to `/tmp` so fresh agent (Codex, Cursor, other Claude session, subagent) picks up work with zero prior context. |

## Layout

```
skills/
  <name>/
    SKILL.md
```

One skill per dir. `SKILL.md` required. Extra files beside it.

## Add skill

```bash
mkdir -p skills/<name>
npx skills init <name>
```

Frontmatter required:

```yaml
---
name: <name>
description: one line trigger
---
```

## Cross-agent

Work with Claude Code, Cursor, Codex, Gemini CLI, OpenCode, Windsurf, 40+ more. `npx skills` resolve path per agent.

## License

MIT
