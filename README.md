# munjez-ci

Shared GitHub Actions for every Munjez app repo in `sijiltech`. App repos call these; they never deploy.

```
app repo push (staging) -> build-and-bump.yml
   build: docker build ONCE -> ghcr.io/sijiltech/<image>:sha-<commit>  (repo GITHUB_TOKEN, packages:write)
   bump : GitHub App token -> commit "image: ghcr.io/sijiltech/<image>@sha256:..." into
          sijiltech/munjez-infra  stacks/staging/<stack>/compose.yaml  (branch main)
munjez-infra push -> webhook -> Komodo staging deploys changed stacks
production: munjez-infra Actions > promote (PR, same digest) -> merge -> Komodo prod Execute (human)
rollback  : git revert the bump/promote commit in munjez-infra
```

## Use it (app repo `.github/workflows/release.yml`)

```yaml
name: release
on:
  push: { branches: [staging] }
permissions: {}
jobs:
  release:
    uses: sijiltech/munjez-ci/.github/workflows/build-and-bump.yml@main
    with: { image: admin-web, stack: admin-web-staging }
    secrets: inherit
    permissions: { contents: read, packages: write }
```

The stack's compose file must already reference `ghcr.io/sijiltech/<image>` (first switch is a manual edit).

## One-time org setup

| What | Where | Value |
|---|---|---|
| GitHub App `sijiltech-deployer` | Org > Settings > Developer settings > GitHub Apps | Repo perms: Contents RW, Pull requests RW, Metadata R. No webhook. Install on **munjez-infra only** |
| `MUNJEZ_DEPLOYER_APP_ID` | Org secret, private repos | App ID from the App page |
| `MUNJEZ_DEPLOYER_PRIVATE_KEY` | Org secret, private repos | App > Generate private key (.pem contents) |
| munjez-ci access | munjez-ci > Settings > Actions > Access | "Accessible from repositories in the sijiltech organization" |
| Package pulls (Komodo) | bot account classic PAT, `read:packages` only | Vault `secret/sijil/staging/komodo` key `GHCR_READ_TOKEN` |

Security: actions pinned by commit SHA; every job starts from `permissions: {}`; the push token never
leaves the build job; the App token is scoped to munjez-infra and lives ~1 h; inputs are regex-checked.
Note: GitHub artifact attestations are not available for private repos on the free plan, so
provenance/SBOM are attached to the image by buildx instead.
