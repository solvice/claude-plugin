# Solvice Claude Code Skills

Five Claude Code skills for building Solvice VRP integrations (three user-invocable, two auto-loaded references).

## Install

Copy this directory to `.claude/skills/solvice/` in your project:

```bash
cp -r claude-plugin/skills/ .claude/skills/solvice/
```

## Skills

| Command | When to use |
|---------|-------------|
| `/solvice:setup` | First time — connects Claude Code to the Solvice MCP |
| `/solvice:scaffold` | Building your first request — generates a working `OnRouteRequest` for your use case |
| `/solvice:debug <id>` | Solution looks wrong — diagnoses constraint violations with specific field fixes |
| `constraint-glossary` | Auto-loaded — full constraint reference used during diagnosis |
| `integration-patterns` | Auto-loaded — canonical request shapes used during scaffolding |

## Requirements

- Claude Code with MCP support
- A Solvice API key (get one at https://app.solvice.io)
