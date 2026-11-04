# Inspect and debug Ravion

Reference file for the [`use-ravion` skill](https://www.ravion.com/SKILL.md). Read it when a run, deploy, or resource is failing, or when you need current state.

```bash
ravion describe <id>                          # any Ravion ID: proj_, env_, minst_, stk_, mdep_, pipe_, prun_, sexec_ — take it from the user's app.ravion.com URL path
ravion environment module-graph <env-id>      # module dependency graph
ravion module list / ravion module get <id>
ravion stack get <id>                         # Terraform state, resources, outputs
ravion deploy list --module-instance-id <id>
ravion module log-sources <module-id>         # runtime log sources and resolved stream prefixes
ravion module logs <module-id> --tail         # application logs; selects the module's source
ravion logs pipeline-step <step-execution-id> --tail    # build/plan/apply output
ravion logs cloudwatch --aws-account-id <id> --log-group-name <group> --region <region> --log-stream-prefix "" --tail
ravion metrics cloudwatch ...
ravion terraform resource list ...            # inspect stack resources
```

Debug order for a failed stack run: `ravion pipeline run get-plan <id>` → `ravion pipeline run get-apply <id>` → `ravion logs pipeline-step <step-execution-id>`.

For a failed deploy: `ravion deploy get <id>` → `ravion module log-sources <module-id>` → `ravion module logs <module-id>`.

## Read application logs

Use CLI 1.26.0 or later for module log commands and the JSON formats below. Upgrade through the [CLI installation guide](https://www.ravion.com/docs/cli/installation) if they are unavailable.

1. Run `ravion module log-sources <module-id> --json`. It returns an array of `{id, name, source}` entries with resolved CloudWatch, EKS Loki, or external-provider configuration. Do not guess a CloudWatch prefix such as `app/` or use pipeline logs as a substitute for application logs.
2. Run `ravion module logs <module-id>`. One source is selected automatically; for several sources, pass `--source <source-id>` from the list. Use `--start-time` and `--end-time` with RFC3339 timestamps to bound a historical query. A source with unresolved stack outputs needs a successful stack apply; an external source gives you its provider URL instead of log events.
3. For analysis, save `ravion module logs <module-id> --json > logs.json`. Successful output is **one JSON array of batches**, not concatenated documents. Parse it with `json.load`, or run `jq '.[].events[]' logs.json`. Inspect all batches; empty batches are valid. Check the command's exit status before using a partial result.
4. For live logs, use `ravion module logs <module-id> --tail --jsonl`. Each line is a compact JSON batch with an `events` array, including empty/final batches. Parse line by line; embedded message newlines are escaped. `--tail --json` is invalid, and `--json` cannot be combined with `--jsonl`. The same format flags apply to the lower-level `ravion logs` commands.

For discovery on older CLIs, `ravion module get <module-id> --json` includes `ui.logs` and `stackId`; `ravion stack get <stack-id> --json` includes `output`. ECS service stacks already expose `log_group_name`, `log_group_arn`, `log_stream_prefix`, and `region`. Prefer the resolved `ui.logs` configuration because other module types use different outputs. When calling `ravion logs cloudwatch` directly, supply either `--log-stream-prefix` or `--log-stream-names`; an explicit empty prefix reads all streams in the selected group.

Runtime logs and build/plan/apply logs are different: use `ravion logs pipeline-step <step-execution-id>` for pipeline output. See the [module logs guide](https://www.ravion.com/docs/modules/logs) for examples.

A stuck Terraform state lock is [documented here](https://www.ravion.com/docs/troubleshooting/stuck-terraform-state-lock); `ravion stack get-lock` and `ravion stack unlock` handle it.

Never run `terraform apply` against a Ravion stack. Stacks change only through their pipelines.

When you report what you found — the failing run, the step whose logs you read, the deploy that is stuck — link each ID as `[<id>](https://app.ravion.com/go/<id>)` so the user can open it in the dashboard.
