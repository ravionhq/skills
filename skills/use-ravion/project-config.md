# Ravion project config workflow

Reference file for the [`use-ravion` skill](https://www.ravion.com/SKILL.md). Read it before creating or changing any infrastructure.

A project config file (`.yaml`, `.jsonc`, or `.cue`) declares environments and their module instances. It is the way to create and change infrastructure. Follow this sequence every time.

```bash
ravion project config schema                     # 1. field-level schema for the file
ravion module definition list                    # 2. available module types
ravion module schema <module-type> [version]     # 3. inputs for each module you touch
ravion project config apply <project-id> --file ravion.yaml --dry-run   # 4. preview
ravion project config apply <project-id> --file ravion.yaml             # 5. apply
```

Generate the file with the CLI so it carries the current header, comments, and links — never hand-write it from memory:

```bash
ravion project create --given-id <project-id> --name "<Project name>" --file ravion.yaml   # new project
ravion project config pull <project-id> --file ravion.yaml                                 # existing project
```

## Module set

List definitions with `ravion module definition list`, then read the input schema of every module you touch with `ravion module schema <module-type> [version]`.

A typical containerized web app is `rvn-aws-network` → `rvn-ecs-cluster` → `rvn-ecs-web`; a static site is `rvn-aws-static`. Add `rvn-rds` or `rvn-aurora` for Postgres/MySQL, `rvn-elasticache` for Redis, `rvn-s3` for buckets, `rvn-lambda` for functions, `rvn-ecs-worker` for background workers. Full list: [module catalog](https://www.ravion.com/docs/module-definitions/catalog).

## File shape

Every environment the user talks about (`production`, `staging`, `preview`) is its own entry under `environments`, and `--environment-given-id staging` targets one of them. Modules wire together by `moduleGivenIdRef` — `{moduleGivenIdRef: "module"}` in the same environment, `"environment.module"` across environments, `"project.environment.module"` across projects.

A database module takes `network: {moduleGivenIdRef: network}`, the same network the cluster uses. The service does **not** get a typed reference to it: `rvn-ecs-web` reads database credentials through runtime `secrets` (`{name, value_from}` pointing at Secrets Manager or SSM), so create the secret from the database outputs and pass it there. Read `ravion module schema rvn-rds` and `ravion module schema rvn-ecs-web` before wiring either side.

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
  - givenId: staging
    name: Staging
    moduleInstances: [] # same module shape as production, usually smaller sizes
```

Adding a database to an existing environment means adding a module instance to that environment's list, then applying scoped to it: `--environment-given-id staging --module-given-id database`.

## Approval rules

Creating new infrastructure can go straight through with `--autoapprove`, scoped to what you are creating so an unrelated change cannot ride along:

```bash
ravion project config apply <project-id> --file ravion.yaml --autoapprove \
  --environment-given-id <environment> --module-given-id <new-module>
```

Omit `--autoapprove` when the change modifies, replaces, or removes existing infrastructure, so a human reviews each Terraform plan first. Removing a module instance plans a destroy and waits for manual approval.

Pass `--description "<PR or commit link>"` to label every stack run the apply creates. Narrow the target with `--environment-given-id`, `--module-given-id`, `--environment-id`, or `--module-instance-id`.

## Follow the stack runs

Every apply prints a `PIPELINE_RUN_ID` per stack run it started, in dependency order, plus the exact wait command. Use `wait --watch`, never a polling loop:

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

A non-zero exit from `wait` means the run failed, was cancelled, or was rolled back. Read the plan and the step logs before retrying — see [operations](https://www.ravion.com/skills/use-ravion/operations.md).

Full reference: [project config file](https://www.ravion.com/docs/config-as-code/project-config-file). Compact machine-readable schema: `https://api.ravion.com/projects/config/schema.md`.
