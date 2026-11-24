# Ravion build and deploy pipelines

Reference file for the [`use-ravion` skill](https://www.ravion.com/SKILL.md). Read it when shipping application code, automating deploys, or rolling back.

Once modules exist, define build and deploy workflows for the deployable ones in a pipeline config file (steps, blocks, inputs, variants, triggers).

One pipeline is one workflow: steps that depend on each other through outputs, approvals, or ordering. Give each independently deployable service its own pipeline and its own trigger. Do not put several services into one pipeline as a `parallel` block of `group`s that each build and deploy a different service — nothing downstream needs both, so they are separate pipelines. Keep services together only when a step needs all of them: a shared image build, a shared approval, an integration test across them, or a required rollout order. Use `parallel` for steps inside one workflow that can overlap; `group` is rare, mostly for a multi-step branch inside `parallel`. Never wrap a flat sequence in a `group` just to name it. See [one workflow per pipeline](https://www.ravion.com/docs/concepts/pipelines#one-workflow-per-pipeline).

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

When pipeline config needs a module output, reference it rather than pasting its resolved value: `<< environments.production.modules.web.output.ecr_repository_arn >>`, or `<< projects.shared.environments.production.modules.web.output.ecr_repository_arn >>` across projects. Verify the output key on the producer. Bare `modules.<module>.output.<key>` references are invalid in pipelines because a pipeline is not tied to one environment. See [module outputs in pipeline templating](https://www.ravion.com/docs/pipelines/templating#module-outputs).

Run it and follow it. `--input` takes JSON keyed by the pipeline's own `inputs` ids:

```bash
ravion pipeline list --project-id <project-id>          # find <pipeline-id>
ravion pipeline run create --pipeline-id <pipeline-id> --variant-id production \
  --description "<why>" --input '{"branch": "main"}'
ravion pipeline run wait <pipeline-run-id> --watch
```

`commit` is optional and resolves to the head of the branch, so pass it only to ship a specific older commit.

## Deploy without a pipeline

Deploy code directly to one module, for example from external CI:

```bash
ravion module list --project-id <project-id>            # find <module-instance-id>
ravion deploy create --module-instance-id <module-instance-id> --description "<why>" --inputs '<json>'
ravion deploy wait <deployment-id> --watch
```

The `--inputs` keys come from the module's deploy manager — check `ravion deploy create --help` and the module's schema before passing them. Never guess them.

## Roll back

```bash
ravion module list --project-id <project-id>            # find <module-instance-id>
ravion deploy list --module-instance-id <module-instance-id>  # find the current deployment ID (mdep_...)
ravion deploy rollback-candidates <current-deployment-id>     # earlier deploys you can return to
ravion deploy rollback <candidate-deployment-id>              # creates a new rollback deployment
ravion deploy wait <new-deployment-id> --watch                # use the ID returned by rollback
```

Both rollback commands take deployment IDs (`mdep_...`), not module instance IDs (`minst_...`). Choose a target from `rollback-candidates`, then wait on the new deployment returned by `rollback`, not the target deployment.

Reference: [pipeline config file](https://www.ravion.com/docs/config-as-code/pipeline-config-file), [step types](https://www.ravion.com/docs/pipelines/step-types), [templating](https://www.ravion.com/docs/pipelines/templating).

## GitLab CI jobs on Ravion runners

A GitLab CI job asks for a Ravion runner with one tag in its `tags`, using the GitHub runner label keys **without `id`**. Write it as one tag or as a list with a bare `ravion` tag; both mean the same:

```yaml
default:
  tags: ["ravion/ee=gitlab-ci/type=sandbox/cpu=4/memory=16"]

build:
  tags:
    [ravion, ee=gitlab-ci, type=sandbox, cpu=8, memory=32, interruptible=true]
```

- `ee` (given ID or `ee_…`) and `type` (`sandbox`, `ec2`, `ec2-spot`) are required. Sandboxes need `cpu` and `memory` and take `disk` and `interruptible`. Kept disks don't work for GitLab jobs yet: leave `disk_cache` out or set it to `off`. Any other `disk_cache` value, like `cpu=auto` sizing, fails the job (`GITLAB_RUNNER_TAG_SETTING_UNSUPPORTED`). EC2 needs `instance_size` (several joined with `+`) and takes `disk`.
- Never add `id=`: GitLab hands each runner one job at a time, and a tag with `id` fails the job (`GITLAB_RUNNER_TAG_INVALID`). Use one Ravion tag per job and keep unrelated tags off Ravion jobs, because every tag becomes part of the runner's tag set.
- Tags are case-sensitive. A job's own `tags` replace `default: tags`.
- Connect the group or project in the dashboard under Settings → GitLab Runners (group Owner or project Maintainer). GitLab runners are in early access, so the organization must have them enabled. Add `interruptible=true` only to jobs that are safe to rerun from the start.
- A job Ravion could not run is failed in GitLab with `runner_system_failure` and an explanation in its log. Retry it from GitLab, or add `retry: { when: runner_system_failure }` to the job.
