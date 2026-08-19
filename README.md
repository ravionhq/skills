# Ravion Skills

Agent skills for [Ravion](https://www.ravion.com), following the [Agent Skills](https://agentskills.io/) format. Skills are packaged instructions that teach AI coding agents how to use a product correctly.

## Available skills

### use-ravion

Deploy and operate infrastructure on AWS with Ravion: connect an AWS account, create projects and module instances through a `ravion.yaml` project config file, build and deploy code through pipelines, follow stack runs and approvals, inspect logs and metrics, and wire config changes into CI.

Source: [`skills/use-ravion/SKILL.md`](skills/use-ravion/SKILL.md), mirrored from the hosted copy at [`https://www.ravion.com/SKILL.md`](https://www.ravion.com/SKILL.md).

Skills for Flightcontrol, Ravion's predecessor, live in [`ravionhq/flightcontrol-skills`](https://github.com/ravionhq/flightcontrol-skills).

## Installation

Install every skill in this repository:

```bash
npx skills add ravionhq/skills
```

Install only the Ravion skill from the hosted copy:

```bash
npx skills add https://www.ravion.com/SKILL.md
```

Agents that support Agent Skills discovery find the hosted skill through [`https://www.ravion.com/.well-known/agent-skills/index.json`](https://www.ravion.com/.well-known/agent-skills/index.json).

### Plugins

`plugins/ravion` bundles the Ravion skill with the Ravion Docs MCP server for Claude Code, Cursor, Codex, and Grok. In Claude Code:

```
/plugin marketplace add ravionhq/skills
/plugin install ravion@ravion-skills
```

## Usage

Skills are available once installed, and the agent uses them when it detects a relevant task. You can also point an agent at the hosted skill directly:

> Deploy this project with Ravion: https://www.ravion.com/SKILL.md

Pair the skill with the Ravion Docs MCP server so the agent can search current documentation:

```bash
npx add-mcp https://www.ravion.com/docs/mcp --name ravion-docs
```

## Skill structure

Each skill contains:

- `SKILL.md` — instructions for the agent
- `scripts/` — helper scripts for automation (optional)
- `references/` — supporting documentation (optional)

## License

MIT
