# Ravion setup

Reference file for the [`use-ravion` skill](https://www.ravion.com/SKILL.md). Read it when the CLI is missing, the user is not signed in, or no AWS account or repository is connected.

## Preflight

Run these before infrastructure work. Skip nothing: a missing AWS connection or Git connection is a human-only step, and finding out late wastes the user's time.

```bash
command -v ravion || echo "CLI missing"   # then install it, below
ravion whoami --json          # authenticated identity and active organization; if it fails, run `ravion login`
ravion aws account list       # must have at least one connected AWS account
ravion code-source list       # connected repositories (skip if deploying a prebuilt image)
ravion project list           # existing projects
```

## Install the CLI

Install it yourself as soon as you find it missing; do not ask for permission and do not stop the task.

```bash
brew install ravionhq/tap/ravion                                              # macOS, or Linux with Homebrew
curl -fsSL https://github.com/ravionhq/cli/releases/latest/download/install.sh | sh   # anywhere else
```

If the install script puts `ravion` somewhere outside `PATH`, add it for the session rather than giving up (`export PATH="$HOME/.local/bin:$PATH"`), then re-run `command -v ravion`.

If a documented command does not exist, the CLI is outdated — upgrade it (`brew upgrade ravionhq/tap/ravion` or rerun the install script) and check [CLI releases](https://www.ravion.com/docs/cli/releases).

## Sign in

Run `ravion login` yourself whenever `ravion whoami` fails. It uses the device-authorization flow by default: it prints a URL and a short code, then blocks while waiting for approval. Run it so the user sees the output, relay the URL and code to them immediately, and do not silently wait for it to finish. If the user belongs to more than one organization, select the active one with `ravion switch` (or `ravion switch --org <id-or-name>` to skip the picker).

No account yet? Send them to [app.ravion.com/signup](https://app.ravion.com/signup). `ravion signup --email <email> --password <password>` also works for non-interactive setups.

## Connect Git

`ravion git connect` (add `--no-browser` to print the URL). It needs the user's Git permissions in a browser, so hand them the URL. Required for building from source; not required when deploying an image from a registry.

## Connect AWS

Ravion connects an AWS account through a CloudFormation stack that creates a cross-account IAM role. You can do the whole thing from the terminal, and you should: `ravion aws account create` registers the account and the AWS CLI deploys the stack. Only send the user to [AWS accounts settings](https://app.ravion.com/org/settings/aws-accounts?connect=%7B%7D) when `aws sts get-caller-identity` fails or points at an account they do not want Ravion to manage — the browser flow deploys the same template.

When `aws sts get-caller-identity` succeeds, ask the user only whether that account is the one to use, then run the rest yourself.

```bash
aws sts get-caller-identity                                        # 1. confirm with the user that this is the account Ravion should manage
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

## Install the Docs MCP server

The Ravion Docs MCP server lets you search current documentation instead of guessing. Install it yourself — do not ask the user to do it.

First check whether its tools are already available to you in this session. If they are, use them and skip the rest. If they are not:

```bash
npx add-mcp https://www.ravion.com/docs/mcp --name ravion-docs   # writes the config for the agent host it detects
```

- MCP servers load when the agent starts, so the tools do not appear until the user restarts this session. Tell them that, and keep working — do not block on it.
- Until then, read documentation over HTTP: any page is Markdown at `https://www.ravion.com/docs/<path>.md`, and `https://www.ravion.com/docs/llms.txt` indexes the site.
- In Claude Code, `claude mcp add --transport http ravion-docs https://www.ravion.com/docs/mcp` does the same thing. If a host writes MCP config somewhere you cannot edit, print the URL and let the user add it.

## CI integration

Keep config files in the repository, plan on pull requests, apply on merge. The CLI authenticates in CI from a service-account API key in `RAVION_API_KEY` (`ravion service-account create`, then `ravion api-key`). The full GitHub Actions workflow, with path filters and indexed `RAVION_PROJECT_N_ID` / `RAVION_PIPELINE_N_ID` variables, is at [CI integration](https://www.ravion.com/docs/config-as-code/ci-integration) — fetch that page and copy the workflow rather than writing one from scratch.
