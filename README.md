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
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: terraform-${{ github.ref }}
  cancel-in-progress: false

jobs:
  ci:
    uses: Tozapid/demo-ci-cd-skeleton/.github/workflows/terraform.yml@main
    permissions:
      contents: read
      id-token: write
```

## What it does

| Job | When | What |
|---|---|---|
| Terraform format and validate | always | `terraform fmt -check -recursive`, `terraform validate` |
| Lambda tests | always | `python -m unittest discover -p '*_test.py'` in `lambda/` |
| Frontend tests | always | `node --check` of `web/*.js` and `cloudfront/*.js`; `npm test` in `tests/web` with Chrome, if the project has it |
| Plan | pull requests from the repository itself | `terraform plan` in the environment `demo`, summary in the run |
| Apply | push to `main`, or a manual run from any branch | plan and apply in the environment `demo`; stops if a bucket, the table or the user pool would be destroyed or replaced |

A manual run (Actions → terraform → Run workflow) deploys the branch picked there over the same site; the next push to `main` puts main back. Applies queue across branches.

The Terraform state is `s3://$TF_STATE_BUCKET/<repository name>/terraform.tfstate`. The site address comes from the output `site_url` or `site_urls`.

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
2. In the GitHub repository: the environment `demo`, the variables `AWS_ROLE_ARN` and `TF_STATE_BUCKET`. No secrets.
3. The ruleset on `main` requires the checks `ci / Terraform format and validate`, `ci / Lambda tests` and `ci / Frontend tests` (the caller's job, then the job here).

## Changing it

A change on `main` here reaches every project on its next run. Check it on a branch first: point one project's caller at `@<branch>` in a pull request.
