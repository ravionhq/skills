# Agent Skills Repository

This repository contains agent skills for Ravion, following the [Agent Skills](https://agentskills.io/) format.

## Available Skills

Skills are located in the `skills/` directory:

- `skills/use-ravion/` - Deploying and operating infrastructure on AWS with Ravion

Each skill has a `SKILL.md` file with instructions and trigger conditions. Skills for Flightcontrol, Ravion's predecessor, live in `ravionhq/flightcontrol-skills`.

## Plugin packaging

`plugins/ravion/` packages the Ravion skill plus the Ravion Docs MCP server for Claude Code, Cursor, Codex, and Grok. `plugins/ravion/skills/use-ravion` is a symlink to `skills/use-ravion`, so the plugin and the `npx skills add` install path always ship the same skill files. Marketplace manifests live in `.claude-plugin/marketplace.json` and `.cursor-plugin/marketplace.json`, and each host's plugin manifest lives under `plugins/ravion/.<host>-plugin/`. Bump `version` in every plugin manifest together.

## Editing skills

- `SKILL.md` frontmatter requires `name` (lowercase, hyphens, max 64 characters) and `description` (max 1024 characters). Keep the description trigger-rich: it is the only text an agent reads when deciding whether to load the skill.
- Keep instructions authoritative and verifiable. Every command must exist in the product's CLI, and every URL must resolve.
- `skills/use-ravion/SKILL.md` is a mirror. Its source of truth is `packages/web/public/SKILL.md` in the website repository, served at `https://www.ravion.com/SKILL.md` and listed in `https://www.ravion.com/.well-known/agent-skills/index.json`. Edit it there; `.github/workflows/sync-hosted-skill.yml` opens a pull request here whenever the hosted copy changes.
