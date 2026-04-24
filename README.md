# gablas-skills

Personal agent skills library. Installable via [skills.sh](https://skills.sh) / `npx skills`.

## Install

All skills:

```bash
npx skills add Gablas/gablas-skills
```

Single skill:

```bash
npx skills add Gablas/gablas-skills --skill learn-source
```

Global (all projects):

```bash
npx skills add Gablas/gablas-skills -g
```

Specific agent:

```bash
npx skills add Gablas/gablas-skills -a claude-code
```

## Skills

| Name | Description |
|------|-------------|
| [`learn-source`](skills/learn-source/SKILL.md) | Research an external API / SaaS / webhook source, produce a ground-truth `docs/sources/<source>/` reference bundle via parallel docs research + live sandbox probes. |

## Layout

```
skills/
  <skill-name>/
    SKILL.md       # name + description frontmatter + instructions
```

Each skill directory = one skill. `SKILL.md` required. Extra files (scripts, templates) live beside it.

## Add a skill

```bash
mkdir -p skills/<name>
npx skills init <name>    # scaffold SKILL.md template
```

Frontmatter:

```yaml
---
name: <name>
description: one-line trigger/purpose
---
```

## Cross-agent

Works with Claude Code, Cursor, Codex, Gemini CLI, OpenCode, Windsurf, + 40 more. Install paths per agent resolved by `npx skills`.

## License

MIT
