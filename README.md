# demo-ci-cd-skeleton

One CI and deployment workflow for the demo projects on AWS:

- [Hide-My-Email](https://github.com/Tozapid/Hide-My-Email)
- [inventory-serverless-aws](https://github.com/Tozapid/inventory-serverless-aws)
- [wiki-serverless-aws](https://github.com/Tozapid/wiki-serverless-aws)

The jobs live here, in [`.github/workflows/terraform.yml`](.github/workflows/terraform.yml), a reusable workflow (`workflow_call`). Each project keeps a short caller, the same file in all of them:

```yaml
name: terraform

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      preview:
        description: "Pull request number: deploy its preview (empty: deploy the branch picked above over the site)"
        type: string
        default: ""
      destroy:
        description: "Destroy that preview instead"
        type: boolean
        default: false
  schedule:
    - cron: "40 4 * * *"

permissions:
  contents: read

concurrency:
  group: terraform-${{ github.event_name == 'schedule' && 'sweep' || github.ref }}-${{ inputs.preview }}
  cancel-in-progress: false

jobs:
  ci:
    uses: Tozapid/demo-ci-cd-skeleton/.github/workflows/terraform.yml@main
    with:
      preview: ${{ inputs.preview || '' }}
      destroy: ${{ inputs.destroy || false }}
    permissions:
      contents: write
      id-token: write
      pull-requests: write
```

## What it does

| Job | When | What |
| --- | --- | --- |
| Terraform format and validate | always | the newest `CHANGELOG.md` entry starts with `## vX.Y.Z`; `terraform fmt -check -recursive`, `terraform validate` |
| Lambda tests | always | `python -m unittest discover -p '*_test.py'` in `lambda/` |
| Frontend tests | always | `node --check` of `web/*.js` and `cloudfront/*.js`; `npm test` in `tests/web` with Chrome, if the project has it |
| Plan | pull requests from the repository itself | `terraform plan` of the site in the environment `demo`, summary in the run |
| Preview | a manual run with `preview` = the number of an open pull request of the repository itself | the stack once more as `<name>-pr<number>`, from the pull request's last push, with its own state; its address, or the failure, in a comment on the pull request |
| Previews to destroy, Destroy preview | the pull request merged or closed; a manual run with `preview` and `destroy`; the daily sweep | `terraform destroy` of the preview, then its state; the comment says so, or that it failed. The sweep takes the previews of pull requests no longer open and those not deployed for `PREVIEW_DAYS` (7), and drops state locks older than `LOCK_HOURS` (3) |
| Apply | push to `main`, or a manual run from any branch | plan and apply in the environment `demo`; stops if a bucket, the table or the user pool would be destroyed or replaced |
| Release | push to `main` or a manual run on `main`, after the apply | the tag and the GitHub release of the newest `CHANGELOG.md` entry, with its text as the notes, unless that version is out already |

A manual run (Actions → terraform → Run workflow) with `preview` empty deploys the branch picked there over the same site; the next push to `main` puts main back. Applies queue across branches.

The Terraform state is `s3://$TF_STATE_BUCKET/<repository name>/terraform.tfstate`, a preview's `<repository name>/pr-<number>/terraform.tfstate`. The site address comes from the output `site_url` or `site_urls`.

### Previews

Nothing is made by itself: Actions → terraform → Run workflow, `preview` = the number of the pull request. The run checks that the pull request is open and not from a fork (a fork's code would run with the deploy role), deploys its last push and comments the address on it; run it again after new pushes. A preview is the stack with `TF_VAR_name=<name>-pr<number>`, `<name>` being the default of the variable `name` in `variables.tf`. Every resource name of the stack has to come from `name`, so that the preview and the site live side by side, and the deploy role, limited to names starting with `<name>`, covers both. It sends no access logs.

A preview goes when its pull request is merged or closed, by hand (`preview` and `destroy`), or with the daily sweep. A failed apply leaves what it made in the state, and the next run or the destroy picks it up. A failed destroy leaves the preview running and says so in the comment; the sweep tries again every day, and "Re-run failed jobs" does at once. A run that died can leave the state locked; the sweep drops a lock older than three hours. A destroy has to work unattended: buckets with data need `force_destroy` in the demo, and a resource with `prevent_destroy` is dropped from the state first only when it lives there alone (`random_*`, `tls_*`, `time_*`); any other one stops the destroy, and the preview is removed by hand. Each preview costs what a demo costs (cents a month) while it lives.

## What a project needs

1. In AWS, the GitHub OIDC provider (`https://token.actions.githubusercontent.com`, audience `sts.amazonaws.com`) and a role for the project that trusts only its environment `demo`:

   ```json
   {
     "Effect": "Allow",
     "Principal": { "Federated": "arn:aws:iam::<account>:oidc-provider/token.actions.githubusercontent.com" },
     "Action": "sts:AssumeRoleWithWebIdentity",
     "Condition": {
       "StringEquals": {
         "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
         "token.actions.githubusercontent.com:sub": "repo:<owner>/<repository>:environment:demo"
       }
     }
   }
   ```

   The token's `sub` names the calling repository, not this one, so each project keeps its own role.
2. In the GitHub repository: the environment `demo`, the variables `AWS_ROLE_ARN` and `TF_STATE_BUCKET`, and optionally `ACCESS_LOGS_BUCKET`: plan and apply pass it to the stack as `TF_VAR_access_logs_bucket`, the bucket the site's CloudFront access logs go to (unset: none). No secrets.
3. A `CHANGELOG.md` whose entries start with their version, newest first: `## v1.4.0`, optionally followed by a date or a title (`## v1.4.0 — 2026-10-12`, `## v1.4.0 — versions`). The sites take their version from the newest entry and compare it with the releases.
4. The caller listens to `pull_request` with `closed` among its types (the preview goes when the pull request does) and to `schedule` (the sweep), takes the inputs `preview` and `destroy` of `workflow_dispatch` and passes them on, and its job grants `contents: write` (for the release job), `id-token: write` (for the OIDC token) and `pull-requests: write` (for the comment of the preview).
5. The ruleset on `main` requires the checks `ci / Terraform format and validate`, `ci / Lambda tests` and `ci / Frontend tests` (the caller's job, then the job here).

## Changing it

A change on `main` here reaches every project on its next run. Check it on a branch first: point one project's caller at `@<branch>` in a pull request.
