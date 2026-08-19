# Inspect and debug Ravion

Reference file for the [`use-ravion` skill](https://www.ravion.com/SKILL.md). Read it when a run, deploy, or resource is failing, or when you need current state.

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

Debug order for a failed stack run: `ravion pipeline run get-plan <id>` → `ravion pipeline run get-apply <id>` → `ravion logs pipeline-step <step-execution-id>`.

For a failed deploy: `ravion deploy get <id>` → the module's CloudWatch logs.

A stuck Terraform state lock is [documented here](https://www.ravion.com/docs/troubleshooting/stuck-terraform-state-lock); `ravion stack get-lock` and `ravion stack unlock` handle it.

Never run `terraform apply` against a Ravion stack. Stacks change only through their pipelines.
