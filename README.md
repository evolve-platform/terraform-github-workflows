# Repository file & setting sync

This repository contains reusable workflows & files for our infrastructure repositories.
These workflows are to be used as building blocks for our GH Actions pipelines. Other files like CODEOWNERS and pre-commit hooks are also synced to repositories.

## OpenTofu plan and apply

`.github/workflows/opentofu-plan-apply-v1.yaml` plans a site's infrastructure
with OpenTofu on a pull request and applies it from the default branch. It runs
every command as a task in the site's own Taskfile, with `TF_CLI=tofu`, so CI
runs what a laptop runs. A site calls it with `uses:`, pinned by commit sha,
once per environment:

```yaml
on:
  pull_request:
  push:
    branches: [main]

jobs:
  infrastructure:
    uses: evolve-platform/terraform-github-workflows/.github/workflows/opentofu-plan-apply-v1.yaml@<commit sha>
    permissions:
      contents: read
      id-token: write
      pull-requests: write
    with:
      environment: tst
      mode: ${{ github.event_name == 'pull_request' && 'plan' || 'apply' }}
      cloud: gcp
      working-directory: configuration
      init-task: init ENV=tst
      plan-task: plan ENV=tst
      apply-task: apply ENV=tst
      gcp-workload-identity-provider: projects/123/locations/global/workloadIdentityPools/github/providers/github
      gcp-plan-service-account: tofu-plan@evolve-tst.iam.gserviceaccount.com
      gcp-apply-service-account: tofu-apply@evolve-tst.iam.gserviceaccount.com

  deploy:
    needs: infrastructure
    # …
```

- **mode.** `plan` runs the init and plan tasks and writes the plan to the job
  summary. Unless `comment: false`, it also writes the plan to one comment per
  environment on the pull request, updated on each push. A failing plan fails
  the check. `apply` runs only on the default branch. It runs the apply task,
  which plans afresh from that branch and applies, so a plan made on a pull
  request is never applied.
- **Tasks.** `init-task`, `plan-task` and `apply-task` default to
  `<environment>:init`, `:plan` and `:apply`. Each takes arguments, such as
  `plan ENV=tst`. A task must run the command in `TF_CLI` and pass the
  workflow's flags on as `CLI_ARGS`, which come after `--`. The plan task must
  hand them to its `plan` (the workflow adds `-out=tfplan`), and the apply task
  to its `apply` (the workflow adds `-auto-approve`).
- **Identities.** No long-lived secrets: the job signs in through OIDC.
  - On `gcp`, through the workload identity provider, as
    `gcp-plan-service-account` or `gcp-apply-service-account`.
  - On `azure`, as `azure-plan-client-id` or `azure-apply-client-id` in
    `azure-tenant-id` and `azure-subscription-id`. OpenTofu reads that
    identity through `ARM_USE_OIDC`.
  - A plan runs without a GitHub environment, so its token's subject is
    `repo:<owner>/<repo>:pull_request`.
  - An apply runs in the GitHub environment named `environment`, where `prd`
    can require an approval. Its token's subject is
    `repo:<owner>/<repo>:environment:<environment>`.
  - Trust each identity for its own subject only. Keep the plan identity
    read-only, except for the state lock and the sops key.
- **sops.** The workflow installs sops and checks it against `sops-sha256`.
  sops decrypts with the signed-in identity, through Cloud KMS on `gcp` and Key
  Vault on `azure`, so that identity needs decrypt rights on the key.
- **Secrets in the plan.** The posted plan is the output of `tofu show`, which
  prints a sensitive value as `(sensitive value)`. A secret appears in the plan
  only where the configuration does not mark it sensitive.
- **Concurrency.** For one repository and environment, applies run one at a
  time and are never cancelled. A newer push to a pull request replaces its
  running plan. The state lock keeps a plan and an apply from overlapping.
