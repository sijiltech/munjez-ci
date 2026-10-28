# munjez-ci

Shared GitHub Actions for every Munjez app repo, wherever it lives. Repos stay in their owners'
accounts; this repo is **public** so any account can call it. It holds workflow code only, no secrets.
App repos call these; they never deploy.

```
app repo push (staging) -> build-and-bump.yml
   build: docker build ONCE -> ghcr.io/<repo owner>/<image>:sha-<commit>  (repo GITHUB_TOKEN, packages:write)
   bump : GitHub App token -> commit "image: ghcr.io/<owner>/<image>@sha256:..." into
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
    with: { image: admin-web, stack: admin-web-staging }   # runs-on: '["self-hosted","Windows"]' if hosted minutes are out
    secrets:   # repos in other accounts cannot use `secrets: inherit`
      MUNJEZ_DEPLOYER_APP_ID: ${{ secrets.MUNJEZ_DEPLOYER_APP_ID }}
      MUNJEZ_DEPLOYER_PRIVATE_KEY: ${{ secrets.MUNJEZ_DEPLOYER_PRIVATE_KEY }}
    permissions: { contents: read, packages: write }
```

Repo keeps its own build job? Call `gitops-bump.yml` with `ref: ghcr.io/<owner>/<image>@sha256:...` instead
(see the header of that file).

The stack's compose file in munjez-infra must already have an `image: ghcr.io/<any owner>/<image>` line.

## One-time setup (free plan: secrets are per repo; org secrets do not reach private repos)

| What | Where | Value |
|---|---|---|
| GitHub App `sijiltech-deployer` | sijiltech > Settings > GitHub Apps | Contents RW, Pull requests RW, Metadata R. No webhook. Installed on **munjez-infra only** |
| `MUNJEZ_DEPLOYER_APP_ID` | Repo secret: each app repo + munjez-infra | App ID (not secret) |
| `MUNJEZ_DEPLOYER_PRIVATE_KEY` | Repo secret: each app repo + munjez-infra | App private key; one key per account owner, each revocable alone |
| Package pulls (Komodo) | classic PAT, `read:packages` only | Vault `secret/sijil/staging/komodo`; package owners grant it Read (Package settings > Manage access) |

Security: actions pinned by commit SHA; every job starts from `permissions: {}`; the push token never
leaves the build job; the App token is scoped to munjez-infra and lives ~1 h; inputs are regex-checked.
GitHub artifact attestations are not available for private repos on the free plan, so provenance and
SBOM are attached to the image by buildx instead.
