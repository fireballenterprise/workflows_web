# Fireball Enterprise — Web / Landing Workflows

Shared GitHub Actions reusable workflows for the family's **static landing sites** (Vite build →
S3 → CloudFront): `fireball_enterprise_landing`, `fireball_sidecar_landing`. Peer to
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
| `deploy.yml` | `env` (dev\|prd), `single_branch` (false), `sync_package_json` (false), `node_version`, `aws_region`, `aws_role_dev`, `aws_role_prd` (req), `stack_dev`, `stack_prd` (req) | bump VERSION (dev / single-branch), build, assume the OIDC role, read `BucketName`/`DistributionId` from the CFN stack, `aws s3 sync dist/ --delete`, `aws cloudfront create-invalidation "/*"`. A single-branch site also tags + publishes the Release here. |
| `release.yml` | `bump` (none), `node_version`, `aws_region`, `aws_role_prd` (req), `stack_prd` (req) | two-branch sites: `resolve_version` → `promote` (development → main) → `deploy` prd → GitHub Release |

## Caller requirements
- **Branches**: single-branch site = `main` only; two-branch site = `development` (working) +
  `main` (production).
- **Deploy auth**: AWS OIDC only — the caller passes IAM role ARNs + CFN stack names as `with:`
  inputs. **No secrets or variables.** The CFN stack must expose `BucketName` + `DistributionId`
  outputs and its deploy role must allow `cloudformation:DescribeStacks`.
- **Git auth**: landing repos have no branch protection → the version bump commits with the
  default `GITHUB_TOKEN`, authored `Levon Becker <LevonBecker@users.noreply.github.com>`. Do not
  set `vars.BOT_APP_ID`.
- **Files**: `package.json` with a `build` script producing `dist/`, `VERSION` (plain `X.Y.Z`),
  and the thin caller workflows below.

## Thin callers

`fireball_enterprise_landing` (single-branch):
```yaml
# .github/workflows/deploy.yml
on: {workflow_dispatch: {}, push: {branches: [main], paths-ignore: [VERSION]}}
jobs:
  deploy:
    permissions: {contents: write, id-token: write}
    uses: fireballenterprise/workflows_web/.github/workflows/deploy.yml@v1
    with:
      env: prd
      single_branch: true
      sync_package_json: true
      aws_role_prd: arn:aws:iam::213533013207:role/fireball-enterprise-landing-deploy
      stack_prd: fireball-landing-prd
    secrets: inherit
```

`fireball_sidecar_landing` (two-branch) additionally has a `release.yml` caller of
`release.yml@v1`.
