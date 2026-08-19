---
name: use-ravion
description: >
  Deploy and operate infrastructure on AWS with Ravion: sign up or sign in,
  connect an AWS account and Git repository, create projects and environments,
  add module instances (VPC, ECS services, RDS, S3, CloudFront, Lambda,
  Terraform stacks) through a project config file, build and deploy application
  code through pipelines, follow stack runs and approvals, inspect logs and
  metrics, roll back, and wire Ravion config changes into CI. Use this skill
  whenever the user mentions Ravion, `ravion.yaml`, a Ravion project, module,
  stack, pipeline, or deploy, or asks to deploy this app or repo to their own
  AWS account, provision AWS infrastructure as code, migrate from Flightcontrol,
  Heroku, Vercel, Railway, or Render to AWS, or debug a failed Ravion deploy,
  Terraform plan, or stack run — even if they don't say "Ravion" explicitly.
license: MIT
allowed-tools: Bash(ravion:*), Bash(aws:*), Bash(brew:*), Bash(curl:*), Bash(npx:*), Bash(git:*), Bash(command:*), Bash(which:*)
metadata:
  author: Ravion
  version: "2.0.0"
  homepage: "https://www.ravion.com/docs"
---

# Use Ravion

Ravion provisions and operates infrastructure in **the user's own AWS account**. Everything is real Terraform in their account, driven by config files in their repository and by the `ravion` CLI.

The CLI is the authoritative interface. Never hand-write Ravion config from memory, and never guess a field: generate config with the CLI and read schemas with the CLI.

## Resource model

| Concept      | What it is                                                                                                      |
| ------------ | --------------------------------------------------------------------------------------------------------------- |
| **Module**   | The thing you create: a VPC, a database, a web service. An instance of a versioned definition from the catalog. |
| **Stack**    | The Terraform state behind a module — the record of the AWS resources it provisions.                            |
| **Deploy**   | A release of application code to a module's runtime, such as a new image on an ECS service.                     |
| **Pipeline** | A workflow of steps — build, deploy, Terraform plan/apply, approvals — that does the work.                      |

Hierarchy: `Organization → Project → Environment → Module instance`. Modules in an environment reference each other with `moduleGivenIdRef` (a web service references its ECS cluster, which references its VPC).

Two independent tracks of change, which is the key mental model:

- Changing a module's **config** (CPU, env var, subnet) starts a **stack change pipeline run**: Terraform plan → approval → apply. No code is rebuilt.
- Shipping **code** runs a **build and deploy pipeline**: build → deploy. No Terraform runs.

## Route by intent

Read the one file you need, when you need it. Each is a URL you can fetch; installed copies keep the same file names next to this one.

| The user wants                                   | Read                                                                                                |
| ------------------------------------------------ | --------------------------------------------------------------------------------------------------- |
| "Deploy this project/repo to AWS with Ravion"    | [Deploy this project](#deploy-this-project) below, which pulls in the files as it goes              |
| Add, change, or remove infrastructure            | [`skills/use-ravion/project-config.md`](https://www.ravion.com/skills/use-ravion/project-config.md) |
| Build and ship code, automate deploys, roll back | [`skills/use-ravion/pipelines.md`](https://www.ravion.com/skills/use-ravion/pipelines.md)           |
| Inspect state, debug a failure, read logs        | [`skills/use-ravion/operations.md`](https://www.ravion.com/skills/use-ravion/operations.md)         |
| Install the CLI, sign in, connect AWS or Git, CI | [`skills/use-ravion/setup.md`](https://www.ravion.com/skills/use-ravion/setup.md)                   |

## Preflight

Always run this first. If anything is missing or unauthenticated, read [setup.md](https://www.ravion.com/skills/use-ravion/setup.md) before continuing — a missing AWS or Git connection needs the user, and finding out late wastes their time.

```bash
command -v ravion || echo "CLI missing"
ravion whoami --json          # authenticated identity and active organization
ravion aws account list       # must have at least one connected AWS account
ravion code-source list       # connected repositories (skip if deploying a prebuilt image)
ravion project list           # existing projects
```

Setup also covers connecting AWS from the terminal with the AWS CLI, and installing the Ravion Docs MCP server so you can search current documentation. Install that server yourself.

## Deploy this project

Golden path for "deploy this repo to AWS with Ravion". Do the work yourself — read the repo, generate the config, dry run, ask, apply — instead of handing the user a list of docs.

1. **Preflight** above.
2. **Detect the app.** Read the manifest (`package.json`, `pyproject.toml`, `go.mod`, `Gemfile`, `Dockerfile`), the build and start commands, the listening port, and whether it is server-rendered or fully static. In a monorepo, find the app root.
3. **Read the framework guide.** `https://www.ravion.com/docs/deploy/aws/<framework>` has a working `ravion.yaml` for Next.js, Astro, Django, Rails, Laravel, FastAPI, SvelteKit, Remix, Vite, and more — see [the index](https://www.ravion.com/docs/deploy/aws). Use it as the source of truth for that stack instead of inventing module inputs.
4. **Create the project and write its config**, following [project-config.md](https://www.ravion.com/skills/use-ravion/project-config.md): generate the file with the CLI, read the schema of every module you touch, then dry run.
5. **Ask before guessing.** AWS account given ID, region, repository slug, domain, instance sizes, and ports are the user's decisions when they cannot be read from the repo with high confidence.
6. **Apply.** New infrastructure can use `--autoapprove`; anything that touches existing infrastructure needs a reviewed plan.
7. **Add a build and deploy pipeline** per [pipelines.md](https://www.ravion.com/skills/use-ravion/pipelines.md) so code ships on every push, then verify the deploy and hand back the service URL. Point a custom domain with [the custom domains guide](https://www.ravion.com/docs/guides/custom-domains).

## Look things up

- Docs MCP server (preferred): search and read pages from your context. Install it per [setup.md](https://www.ravion.com/skills/use-ravion/setup.md).
- Any docs page is readable as Markdown at `https://www.ravion.com/docs/<path>.md`, and the whole site is indexed at `https://www.ravion.com/docs/llms.txt`.
- Module catalog: [`https://www.ravion.com/docs/module-definitions/catalog`](https://www.ravion.com/docs/module-definitions/catalog). CLI reference: [`https://www.ravion.com/docs/cli/overview`](https://www.ravion.com/docs/cli/overview).
- Every CLI command supports `--help`, and most reads support `--json`. Prefer `--json` when you need to parse output.

## Rules

Always:

- Generate config files with `ravion project create --file` or `ravion project config pull --file`, then edit them.
- Check `ravion project config schema` and `ravion module schema <type>` before writing module inputs.
- Dry run every change, and show the user the planned diff before applying.
- Preserve existing IDs, `givenId` values, module versions, and links you did not intend to change.
- Ask before production-impacting changes: public access, deletion protection, backup retention, capacity, region, networking exposure.
- Use `wait --watch` to follow runs and deploys.

Never:

- Invent fields, module types, or versions that are not in the CLI schema output.
- Copy module inputs from unrelated files or examples.
- Apply without a successful dry run.
- Poll `ravion pipeline run get` or `ravion deploy get` in a loop.
- Use `--autoapprove` for a change that touches existing infrastructure.
- Run `terraform apply` against a Ravion stack. Stacks change only through their pipelines.
- Edit legacy Flightcontrol config (`flightcontrol.json`, `flightcontrol.cue`) as if it were Ravion config. To move a project over, follow [migrate from Flightcontrol](https://www.ravion.com/docs/migrate/from-flightcontrol).

## Report bugs and feedback

Report friction, not just breakage. Both commands are non-interactive, exit `0` on success, and are safe to run unattended, so use them as you go rather than saving them for the end.

```bash
ravion report bug "<what happened and what you expected>" --command "<the command that failed>"
ravion report feedback "<what was confusing, harder than it should be, or missing>"
```

Send a bug when a command fails unexpectedly, errors misleadingly, or Ravion behaves differently than documented. Include the exact error text and the IDs of the resources involved.

Send feedback whenever the product got in your way, even though you eventually finished:

- A task took more steps, guesses, or retries than it should have.
- You could not tell which command, module, or field to use, or the docs and CLI disagreed.
- You needed something the CLI or API does not expose, and worked around it in the console, in raw Terraform, or by hand.
- An error message did not say what to do next.
- You had to ask the user for something the tooling could have determined itself.

Say what you were trying to do, what you tried, and what would have made it obvious. This is the main way these rough edges get found, so err on the side of sending it.
