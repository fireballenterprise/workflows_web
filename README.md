# Fireball Enterprise — Web / Landing Workflows

Shared GitHub Actions reusable workflows for the family's **static sites** (Vite build → S3 →
CloudFront): `fireball_designer_web`, `fireball_fabrication_web`, `fireball_filming_web`. Peer to
[`workflows_shopify`](https://github.com/fireballenterprise/workflows_shopify); the shared
version / promote / release primitives live in
[`workflows_common`](https://github.com/fireballenterprise/workflows_common).

## Versioning
`v`-prefix dual tags (`v1.0.0` + floating `v1`), same scheme as `workflows_shopify` /
`workflows_common`. Callers reference `@v1`. Bump `VERSION` in a PR to cut a release.

## Reusable workflows
`uses: fireballenterprise/workflows_web/.github/workflows/<name>.yml@v1`

| workflow | inputs | does |
|---|---|---|
| `tests.yml` | `node_version` (24), `lfs` (true) | `npm ci` + `npm run build` — the PR check |
| `deploy.yml` | `env` (dev\|prd), `single_branch` (false), `sync_package_json` (false), `node_version`, `aws_region`, `aws_account_dev` (726085025386), `aws_account_prd` (213533013207), `aws_role_dev`, `aws_role_prd` (req; a bare role name or a full ARN), `stack_dev`, `stack_prd` (req) | bump VERSION (dev / single-branch), build, assume the OIDC role, read `BucketName`/`DistributionId` from the CFN stack, `aws s3 sync dist/ --delete`, `aws cloudfront create-invalidation "/*"`. A single-branch site also tags + publishes the Release here. |
| `release.yml` | `bump` (none), `node_version`, `aws_region`, `aws_role_prd` (req), `stack_prd` (req) | two-branch sites: `resolve_version` → `promote` (development → main) → `deploy` prd → GitHub Release |

## Caller requirements
- **Branches**: single-branch site = `main` only; two-branch site = `development` (working) +
  `main` (production).
- **Deploy auth**: AWS OIDC only — the caller passes IAM role names (or ARNs) + CFN stack names as `with:`
  inputs. **No secrets or variables.** The CFN stack must expose `BucketName` + `DistributionId`
  outputs and its deploy role must allow `cloudformation:DescribeStacks`.
- **Git auth**: the static-site repos have no branch protection → the version bump commits with the
  default `GITHUB_TOKEN`, authored `Levon Becker <LevonBecker@users.noreply.github.com>`. Do not
  set `vars.BOT_APP_ID`.
- **Files**: `package.json` with a `build` script producing `dist/`, `VERSION` (plain `X.Y.Z`),
  and the thin caller workflows below.

## Thin callers

`fireball_fabrication_web` (two-branch; Filming and Designer are the same shape):
```yaml
# .github/workflows/deploy.yml
on:
  push: {branches: [development]}
  workflow_dispatch: {inputs: {env: {type: choice, default: dev, options: [dev, prd]}}}
jobs:
  deploy:
    permissions: {contents: write, id-token: write}
    if: ${{ vars.ENABLE_CI == 'true' }}
    uses: fireballenterprise/workflows_web/.github/workflows/deploy.yml@v1
    with:
      env: ${{ inputs.env || 'dev' }}
      aws_role_dev: fireball-fabrication-web-dev-deploy      # a bare name: the account comes from aws_account_dev (726085025386)
      aws_role_prd: fireball-fabrication-web-deploy          # prd account (213533013207)
      stack_dev: fireball-fabrication-web-dev
      stack_prd: fireball-fabrication-web-prd
    secrets: inherit
```

Each also has a `tests.yml` caller of `tests.yml@v1` and a `release.yml` caller of
`release.yml@v1`. `single_branch: true` (bump, deploy prd and release from `main` alone) is still
supported but no current site uses it.
