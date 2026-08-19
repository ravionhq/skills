# Ravion build and deploy pipelines

Reference file for the [`use-ravion` skill](https://www.ravion.com/SKILL.md). Read it when shipping application code, automating deploys, or rolling back.

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

## Deploy without a pipeline

Deploy code directly to one module, for example from external CI:

```bash
ravion deploy create --module-instance-id <module-instance-id> --description "<why>" --inputs '<json>'
ravion deploy wait <deployment-id> --watch
```

The `--inputs` keys come from the module's deploy manager — check `ravion deploy create --help` and the module's schema before passing them. Never guess them.

## Roll back

```bash
ravion deploy rollback-candidates <module-instance-id>
ravion deploy rollback <deployment-id>
```

Reference: [pipeline config file](https://www.ravion.com/docs/config-as-code/pipeline-config-file), [step types](https://www.ravion.com/docs/pipelines/step-types), [templating](https://www.ravion.com/docs/pipelines/templating).
