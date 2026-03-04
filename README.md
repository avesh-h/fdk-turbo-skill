# Fynd FDK React Theme Skill

An [Agent Skill](https://agentskill.sh/readme) that gives AI agents context and best practices for the **Fynd FDK React Theme** — a production e-commerce theme on the Fynd Commerce platform.

## What it does

- Teaches agents FPI/FDK-specific patterns, SSR compatibility, GraphQL data fetching, and platform integration rules
- Covers theme structure (pages, sections, components), styling conventions, analytics events, and URL/navigation rules
- Use when writing, reviewing, or refactoring code in a Fynd theme repository

## Install in your agentic IDE

### From agentskill.sh (after the skill is listed)

If the skill is published on [agentskill.sh](https://agentskill.sh/), install with:

```bash
/learn @avesh-h/fynd-theme
```

Or clone into your platform’s skills directory:

| Platform | Command / path |
|----------|-----------------|
| **Cursor** | `git clone https://github.com/avesh-h/fdk-turbo-skill.git ~/.cursor/skills/fynd-theme` |
| **Claude Code** | `git clone https://github.com/avesh-h/fdk-turbo-skill.git ~/.claude/skills/fynd-theme` |
| **Codex** | `git clone https://github.com/avesh-h/fdk-turbo-skill.git ~/.codex/skills/fynd-theme` |
| **Windsurf** | `git clone https://github.com/avesh-h/fdk-turbo-skill.git ~/.windsurf/skills/fynd-theme` |

Restart your agent after installing.

## Repo structure

- **SKILL.md** — Main skill definition (metadata + instructions). Required by all skill-compatible tools.
- **references/** — Topic-specific rule files referenced from `SKILL.md`.

## License

ISC
