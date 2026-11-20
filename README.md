# Repository file & setting sync

This repository contains reusable workflows & files for our infrastructure repositories.
These workflows are to be used as building blocks for our GH Actions pipelines. Other files like CODEOWNERS and pre-commit hooks are also synced to repositories.

## Plan and apply

`.github/workflows/plan-apply-v1.yaml` plans a site's infrastructure on a pull
request and applies it from the default branch, with Terraform or OpenTofu. It
runs the cli directly, so a site needs nothing besides its root module. A site
calls it with `uses:`, pinned by commit sha, once per environment:

```yaml
on:
  pull_request:
  push:
    branches: [main]

jobs:
  infrastructure:
    uses: evolve-platform/terraform-github-workflows/.github/workflows/plan-apply-v1.yaml@<commit sha>
    permissions:
      contents: read
      id-token: write
      pull-requests: write
    with:
      environment: tst
      mode: ${{ github.event_name == 'pull_request' && 'plan' || 'apply' }}
      cloud: gcp
      cli: tofu
      cli-version: 1.13.9
      working-directory: configuration
      gcp-workload-identity-provider: projects/123/locations/global/workloadIdentityPools/github/providers/github
      gcp-plan-service-account: plan@evolve-tst.iam.gserviceaccount.com
      gcp-apply-service-account: apply@evolve-tst.iam.gserviceaccount.com

  deploy:
    needs: infrastructure
    # …
```

- **mode.** `plan` runs `init` and `plan` and writes the plan to the job
  summary. Unless `comment: false`, it also writes the plan to one comment per
  environment on the pull request, updated on each push. A failing plan fails
  the check. `apply` runs only on the default branch. It runs `init` and
  `apply -auto-approve`, which plans afresh from that branch, so a plan made on
  a pull request is never applied.
- **cli.** `terraform` or `tofu`, at `cli-version`.
- **Files.** `init` reads the backend configuration from `backend-config`, and
  `plan` and `apply` read the variables from `var-file`. Both paths are
  relative to `working-directory`. Left empty, they default to
  `envs/<environment>.backend.hcl` and `envs/<environment>.tfvars`, and only
  where those files exist.
- **Identities.** No long-lived secrets: the job signs in through OIDC.
  - On `gcp`, through the workload identity provider, as
    `gcp-plan-service-account` or `gcp-apply-service-account`.
  - On `azure`, as `azure-plan-client-id` or `azure-apply-client-id` in
    `azure-tenant-id` and `azure-subscription-id`. The providers read that
    identity through `ARM_USE_OIDC`.
  - A plan runs without a GitHub environment, so its token's subject is
    `repo:<owner>/<repo>:pull_request`.
  - An apply runs in the GitHub environment named `environment`, where `prd`
    can require an approval. Its token's subject is
    `repo:<owner>/<repo>:environment:<environment>`.
  - Trust each identity for its own subject only. Keep the plan identity
    read-only, except for the state lock and the sops key.
- **sops.** A sops provider decrypts with the signed-in identity, through Cloud
  KMS on `gcp` and Key Vault on `azure`, so that identity needs decrypt rights
  on the key. The workflow installs no sops binary.
- **Secrets in the plan.** The posted plan is the output of `show`, which
  prints a sensitive value as `(sensitive value)`. A secret appears in the plan
  only where the configuration does not mark it sensitive.
- **Concurrency.** For one repository and environment, applies run one at a
  time and are never cancelled. A newer push to a pull request replaces its
  running plan. The state lock keeps a plan and an apply from overlapping.
