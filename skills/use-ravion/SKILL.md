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
  version: "1.0.0"
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

| The user wants                                | Go to                                                     |
| --------------------------------------------- | --------------------------------------------------------- |
| "Deploy this project/repo to AWS with Ravion" | [Deploy this project](#deploy-this-project)               |
| Add, change, or remove infrastructure         | [Project config workflow](#project-config-workflow)       |
| Build and ship code, automate deploys         | [Build and deploy pipelines](#build-and-deploy-pipelines) |
| Inspect state, debug a failure, read logs     | [Inspect and debug](#inspect-and-debug)                   |
| Plan config changes on PRs and apply on merge | [CI integration](#ci-integration)                         |
| Sign up, connect AWS, connect Git             | [Preflight](#preflight)                                   |

## Preflight

Run these before infrastructure work. Skip nothing: a missing AWS connection or Git connection is a human-only step, and finding out late wastes the user's time.

```bash
command -v ravion || echo "CLI missing"
ravion whoami --json          # authenticated identity and active organization
ravion aws account list       # must have at least one connected AWS account
ravion code-source list       # connected repositories (skip if deploying a prebuilt image)
ravion project list           # existing projects
```

**Install the CLI** if it is missing:

```bash
brew install ravionhq/tap/ravion                                              # macOS
curl -fsSL https://github.com/ravionhq/cli/releases/latest/download/install.sh | sh
```

**Sign in.** `ravion login` uses the device-authorization flow by default: it prints a URL and a short code, then blocks while waiting for approval. Run it so the user sees the output, relay the URL and code to them immediately, and do not silently wait for it to finish. If the user belongs to more than one organization, select the active one with `ravion switch` (or `ravion switch --org <id-or-name>` to skip the picker).

No account yet? Send them to [app.ravion.com/signup](https://app.ravion.com/signup). `ravion signup --email <email> --password <password>` also works for non-interactive setups.

**Human-only prerequisites.** These require a browser and the user's cloud and Git permissions. Stop and ask the user to complete the ones that are missing, then continue:

- Connect AWS: [AWS accounts settings](https://app.ravion.com/org/settings/aws-accounts?connect=%7B%7D) runs a CloudFormation stack in their account. If they are already signed in to the AWS CLI for that account, do it from the terminal instead — see [Connect AWS from the terminal](#connect-aws-from-the-terminal).
- Connect Git: `ravion git connect` (add `--no-browser` to print the URL). Needed for building from source; not needed when deploying an image from a registry.

### Install the Docs MCP server

The Ravion Docs MCP server lets you search current documentation instead of guessing. Install it yourself — do not ask the user to do it.

First check whether its tools are already available to you in this session. If they are, use them and skip the rest. If they are not:

```bash
npx add-mcp https://www.ravion.com/docs/mcp --name ravion-docs   # writes the config for the agent host it detects
```

- MCP servers load when the agent starts, so the tools do not appear until the user restarts this session. Tell them that, and keep working — do not block on it.
- Until then, read documentation over HTTP: any page is Markdown at `https://www.ravion.com/docs/<path>.md`, and `https://www.ravion.com/docs/llms.txt` indexes the site.
- In Claude Code, `claude mcp add --transport http ravion-docs https://www.ravion.com/docs/mcp` does the same thing. If a host writes MCP config somewhere you cannot edit, print the URL and let the user add it.

### Connect AWS from the terminal

Ravion connects an AWS account through a CloudFormation stack that creates a cross-account IAM role. The console flow and the AWS CLI flow deploy the same template, so use the CLI whenever the user already has credentials for the target account.

```bash
aws sts get-caller-identity                                        # 1. confirm the credentials point at the account they want Ravion to manage
ravion aws account create --given-id <id> --name "<Name>" --json   # 2. returns the Ravion account id, "aws_..."
ravion aws account cloudformation-template-url <aws-account-id> --json  # 3. returns {"templateUrl": "...", "version": "..."}

# 4. deploy the template, always in us-east-1
aws cloudformation create-stack \
  --region us-east-1 \
  --stack-name ravion-<aws-account-id> \
  --template-url "<templateUrl>" \
  --capabilities CAPABILITY_NAMED_IAM

aws cloudformation wait stack-create-complete --region us-east-1 --stack-name ravion-<aws-account-id>
ravion aws account get <aws-account-id> --json                     # 5. status flips to CONNECTED
```

- Show the user the output of `aws sts get-caller-identity` and confirm the account number before creating the stack. It is the only check that the credentials belong to the account they mean.
- Create the stack in `us-east-1`. Its authenticator custom resource lives there. The role it creates lets Ravion provision in any region.
- Pass no `--parameters`. The template already carries the account id and a single-use onboarding token, so treat the URL as a secret and fetch it right before creating the stack.
- `CAPABILITY_NAMED_IAM` is required: the stack creates a named role and named managed policies. Replace `_` with `-` in the account id when it appears in the stack name.
- Upgrading permissions: when `ravion aws account get` reports a `roleVersion` older than `latestRoleVersion`, show the user `ravion aws account policy-diff <aws-account-id>`, then update the same stack with a freshly fetched template URL — `aws cloudformation update-stack --region us-east-1 --stack-name <name> --template-url "<templateUrl>" --capabilities CAPABILITY_NAMED_IAM`. The `cfnStackId` from step 3 identifies the existing stack when the name is unknown.

## Deploy this project

Golden path for "deploy this repo to AWS with Ravion". Do the work yourself — read the repo, generate the config, dry run, ask, apply — instead of handing the user a list of docs.

1. **Preflight** above. Confirm an AWS account and (for source builds) the repository are connected.
2. **Detect the app.** Read the manifest (`package.json`, `pyproject.toml`, `go.mod`, `Gemfile`, `Dockerfile`), the build and start commands, the listening port, and whether it is server-rendered or fully static. In a monorepo, find the app root.
3. **Read the framework guide.** `https://www.ravion.com/docs/deploy/aws/<framework>` has a working `ravion.yaml` for Next.js, Astro, Django, Rails, Laravel, FastAPI, SvelteKit, Remix, Vite, and more — see [the index](https://www.ravion.com/docs/deploy/aws). Use it as the source of truth for that stack instead of inventing module inputs.
4. **Create the project and its config file.** Generate the file with the CLI so it carries the current header, comments, and links:

   ```bash
   ravion project create --given-id <project-id> --name "<Project name>" --file ravion.yaml
   ```

   For an existing project, pull instead: `ravion project config pull <project-id> --file ravion.yaml`.

5. **Pick the module set** with `ravion module definition list`, then read the input schema of each module before writing it: `ravion module schema <module-type> [version]`. A typical containerized web app is `rvn-aws-network` → `rvn-ecs-cluster` → `rvn-ecs-web`; a static site is `rvn-aws-static`. Add `rvn-rds` or `rvn-aurora` for Postgres/MySQL, `rvn-elasticache` for Redis, `rvn-s3` for buckets, `rvn-lambda` for functions, `rvn-ecs-worker` for background workers.
6. **Ask before guessing.** AWS account given ID, region, repository slug, domain, instance sizes, and ports are the user's decisions when they cannot be read from the repo with high confidence.
7. **Dry run, then apply** — see below. New infrastructure can use `--autoapprove`.
8. **Add a build and deploy pipeline** so code ships on every push, then verify the deploy and hand back the service URL. Point a custom domain with [the custom domains guide](https://www.ravion.com/docs/guides/custom-domains).

## Project config workflow

A project config file (`.yaml`, `.jsonc`, or `.cue`) declares environments and their module instances. It is the way to create and change infrastructure. Follow this sequence every time.

```bash
ravion project config schema                     # 1. field-level schema for the file
ravion module definition list                    # 2. available module types
ravion module schema <module-type> [version]      # 3. inputs for each module you touch
ravion project config apply <project-id> --file ravion.yaml --dry-run   # 4. preview
ravion project config apply <project-id> --file ravion.yaml             # 5. apply
```

Example `ravion.yaml` shape — modules wire together by `moduleGivenIdRef`:

```yaml
project:
  givenId: demo
  name: Demo
environments:
  - givenId: production
    name: Production
    moduleInstances:
      - givenId: network
        name: Network
        type: rvn-aws-network
        version: 1.0.0
        input:
          aws_account_id: ravion-prod
          aws_region: us-east-1
          name: demo-production
      - givenId: cluster
        name: ECS cluster
        type: rvn-ecs-cluster
        version: 1.0.0
        input:
          network:
            moduleGivenIdRef: network
          name: demo-production
      - givenId: web
        name: Web service
        type: rvn-ecs-web
        version: 1.0.0
        input:
          cluster:
            moduleGivenIdRef: cluster
          name: demo-production-web
          build_source: railpack
          source_repo: my-org/my-repo
          container_port: 3000
```

**Approval rules.** Pass `--autoapprove` only when the apply _only creates_ new infrastructure, so its stack runs go straight through:

```bash
ravion project config apply <project-id> --file ravion.yaml --autoapprove
```

Omit `--autoapprove` when the change modifies, replaces, or removes existing infrastructure, so a human reviews each Terraform plan first. Removing a module instance plans a destroy and waits for manual approval.

Pass `--description "<PR or commit link>"` to label every stack run the apply creates. Narrow the target with `--environment-given-id`, `--module-given-id`, `--environment-id`, or `--module-instance-id`.

**Follow the stack runs.** Every apply prints a `PIPELINE_RUN_ID` per stack run it started, in dependency order, plus the exact wait command. Use `wait --watch`, never a polling loop:

```bash
# Without --autoapprove: wait for the gate, read the plan, approve, wait for the apply
ravion pipeline run wait <pipeline-run-id> --until PENDING_APPROVAL --watch
ravion pipeline run get-plan <pipeline-run-id>
ravion pipeline run approve <pipeline-run-id>
ravion pipeline run wait <pipeline-run-id> --watch

# With --autoapprove
ravion pipeline run wait <pipeline-run-id> --watch --timeout 30m
ravion pipeline run get-apply <pipeline-run-id>
```

A non-zero exit from `wait` means the run failed, was cancelled, or was rolled back. Read the plan and the step logs before retrying.

Full reference: [project config file](https://www.ravion.com/docs/config-as-code/project-config-file). Compact machine-readable schema: `https://api.ravion.com/projects/config/schema.md`.

## Build and deploy pipelines

Once modules exist, define build and deploy workflows for the deployable ones in a pipeline config file (steps, groups, inputs, variants, triggers).

```bash
ravion pipeline schema
ravion pipeline create --given-id <pipeline-id> --name "<name>" --project-id <project-id>
ravion pipeline config pull <pipeline-id> --file ravion-pipeline.yaml
ravion pipeline config apply <pipeline-id> --file ravion-pipeline.yaml   # creates a new pipeline version
```

A minimal build-then-deploy pipeline, where the deploy consumes the build's digest and a GitHub push trigger runs it:

```yaml
triggers:
  - id: github_push
    type: webhook:github
    event: push
    repo: https://github.com/my-org/my-repo
    filter:
      branch: main
    run:
      production:
        input:
          branch: << trigger.payload.branch >>
          commit: << trigger.payload.commit >>
inputs:
  - id: branch
    type: string
    default: main
  - id: commit
    type: string
    required: false
variants:
  - id: production
    name: Production
steps:
  - group: Build and deploy
    steps:
      - id: build_web
        name: Build web
        type: build
        module_instance: << pipeline.variant.id >>.web
        input:
          branch: << pipeline.input.branch >>
          ref: << pipeline.input.commit >>
      - id: deploy_web
        name: Deploy web
        type: deploy
        module_instance: << pipeline.variant.id >>.web
        input:
          image_ref: << steps.build_web.output.image_digest >>
```

Run it and follow it:

```bash
ravion pipeline run create --pipeline-id <pipeline-id> --variant-id production --description "<why>"
ravion pipeline run wait <pipeline-run-id> --watch
```

Deploy code directly to one module without a pipeline, for example from external CI:

```bash
ravion deploy create --module-instance-id <module-instance-id> --description "<why>" --inputs '<json>'
ravion deploy wait <deployment-id> --watch
```

The `--inputs` keys come from the module's deploy manager — check `ravion deploy create --help` and the module's schema before passing them. Roll back with `ravion deploy rollback-candidates <id>` then `ravion deploy rollback <id>`. Reference: [pipeline config file](https://www.ravion.com/docs/config-as-code/pipeline-config-file), [step types](https://www.ravion.com/docs/pipelines/step-types), [templating](https://www.ravion.com/docs/pipelines/templating).

## Inspect and debug

```bash
ravion describe <id>                          # any Ravion ID: proj_, env_, mod_, stack_, prun_, dep_
ravion environment module-graph <env-id>      # module dependency graph
ravion module list / ravion module get <id>
ravion stack get <id>                         # Terraform state, resources, outputs
ravion deploy list --module-instance-id <id>
ravion logs pipeline-step <step-execution-id> --tail    # build/plan/apply output
ravion logs cloudwatch --aws-account-id <id> --log-group-name <group> --region <region> --tail
ravion metrics cloudwatch ...
ravion terraform resource list ...            # inspect stack resources
```

Debug order for a failed stack run: `ravion pipeline run get-plan <id>` → `ravion pipeline run get-apply <id>` → `ravion logs pipeline-step <step-execution-id>`. For a failed deploy: `ravion deploy get <id>` → the module's CloudWatch logs. A stuck Terraform state lock is [documented here](https://www.ravion.com/docs/troubleshooting/stuck-terraform-state-lock); `ravion stack get-lock` and `ravion stack unlock` handle it.

Never run `terraform apply` against a Ravion stack. Stacks change only through their pipelines.

## CI integration

Keep config files in the repository, plan on pull requests, apply on merge. The CLI authenticates in CI from a service-account API key in `RAVION_API_KEY` (`ravion service-account create`, then `ravion api-key`). The full GitHub Actions workflow, with path filters and indexed `RAVION_PROJECT_N_ID` / `RAVION_PIPELINE_N_ID` variables, is at [CI integration](https://www.ravion.com/docs/config-as-code/ci-integration) — fetch that page and copy the workflow rather than writing one from scratch.

## Look things up

- Docs MCP server (preferred): search and read pages from your context. See [Install the Docs MCP server](#install-the-docs-mcp-server).
- Any docs page is readable as Markdown at `https://www.ravion.com/docs/<path>.md`, and the whole site is indexed at `https://www.ravion.com/docs/llms.txt`.
- Module catalog: [`https://www.ravion.com/docs/module-definitions/catalog`](https://www.ravion.com/docs/module-definitions/catalog). CLI reference: [`https://www.ravion.com/docs/cli/overview`](https://www.ravion.com/docs/cli/overview).
- Every CLI command supports `--help`, and most reads support `--json`. Prefer `--json` when you need to parse output.
- If a command does not exist, the CLI is likely outdated — upgrade it (`brew upgrade ravionhq/tap/ravion` or rerun the install script) and check [CLI releases](https://www.ravion.com/docs/cli/releases).

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
