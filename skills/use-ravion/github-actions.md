# Move GitHub Actions jobs to Ravion runners

Use this when the user wants their GitHub Actions jobs to run on Ravion runners: "move our CI to Ravion", "switch `runs-on` to Ravion", or "migrate off GitHub-hosted, Blacksmith, Depot, RunsOn, Namespace, BuildJet, WarpBuild, Ubicloud or Cirrus runners".

Ravion's migration pull request makes many changes besides `runs-on`. A hand migration must make the same ones. The full list is [Migrate GitHub Actions workflows](https://www.ravion.com/docs/github-actions/migrate-workflows.md). Read it before editing any workflow, and also read [runners](https://www.ravion.com/docs/github-actions/runners.md), [caching](https://www.ravion.com/docs/github-actions/caching.md), [AWS credentials](https://www.ravion.com/docs/github-actions/aws-credentials.md) and [Amazon ECR](https://www.ravion.com/docs/github-actions/amazon-ecr.md).

## Prerequisites

- The Ravion GitHub App is installed on a GitHub **organization** (not a personal account) and connected to the Ravion organization.
- A dedicated CI execution environment exists. Find it with `ravion execution-environment list`. Ask the user which one to use if it isn't obvious. Never use the environment that runs production, and never invent an ID.
- GitHub Actions runners are in private preview. If jobs never start, the organization may not have access yet; tell the user to [contact support](https://www.ravion.com/docs/contact-support).

## Preferred path: the migration preview

The preview runs the same code as the migration pull request and returns its diff, without opening a pull request. Use it whenever the CLI is signed in and the repository is connected through the GitHub App.

```bash
ravion execution-environment github-runner-migration-repositories <ee_id>      # the repository must be listed
ravion execution-environment start-github-runner-migration-preview <ee_id> --repositories <owner>/<repo> --json
ravion execution-environment github-runner-migration-preview <ee_id> <previewId> --json
```

`<ee_id>` is the environment's unique ID (`ee_…`). Fetch the preview again every few seconds until `status` is `completed` or `failed`; this is a short wait for one background job, not a deploy to poll. In each entry of `results`:

1. Create a branch from `baseSha` and apply `diff` with `git apply`. If the workflows have changed since, make the same edits by hand, using `diff` and `decisions` as the reference.
2. Read `skipped`: those jobs stay on their current runner. Report them with their reasons.
3. Read `pullRequestBody` and do or report its follow-ups: steps that expect an empty workspace, jobs that need AWS credentials for Amazon ECR, and optional speedups.

## Fallback: the checklist

Without a session or a connected repository, follow the checklist in [Migrate GitHub Actions workflows](https://www.ravion.com/docs/github-actions/migrate-workflows.md#checklist) for every job in `.github/workflows`. The steps people most often miss:

- Label every moved job `ravion/ee=<ee>/type=sandbox/cpu=auto/memory=auto/disk=auto/disk_cache=auto/interruptible=<true|false>/id=${{ github.run_id }}-${{ github.run_attempt }}-<workflow-file>-<job-id>`, plus matrix values or `${{ strategy.job-index }}` for matrix jobs.
- Decide `interruptible` with the guide's ordered rules. Every job in a workflow triggered by releases, tags or `release*` branches is not interruptible.
- Swap or remove vendor actions (Blacksmith, Depot, Namespace, BuildJet, WarpBuild, RunsOn, Ubicloud, Cirrus caches, builders and setup actions, and disk-freeing actions). Leave the job alone when a vendor Docker builder can't be removed safely.
- Add `clean: false` to `actions/checkout`, except in jobs that publish, deploy, release, use a GitHub `environment:`, or check for a clean tree.
- Make `mkdir`, `ln -s`, `git remote add`, `git worktree add` and `git clone` work when an earlier run's files are still there.
- Move ARM jobs to sandboxes unless they push container images. Leave macOS, Windows and GPU jobs, and `runs-on` taken from the matrix.

## Rules

- Don't rename moved jobs or workflow files. Automatic sizing matches a job to the runner it last ran on, and learns sizes, by workflow file and job name.
- Preserve triggers, permissions, secrets, matrices, `needs`, and every job ID.
- Don't trigger workflows, merge, or create IAM roles in the user's AWS account without their explicit yes. Creating the OIDC role an ECR job needs is a change in their AWS account; list it as a follow-up if they haven't approved it.
- Report the diff, each job's interruptible decision and why, the jobs left on their current runner and why, and the follow-ups you didn't make.
