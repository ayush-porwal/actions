# actions

These are reusable **GitHub Actions** workflows, called at the _job_ level from a consuming repo.

## Usage

Reference a workflow by its full path plus a git ref:

```yaml
jobs:
  build:
    uses: ayush-porwal/actions/.github/workflows/nodejs-cdk-ci.yaml@main
    with:
      working-directory: infra
      node-version: "24.x"
```

Notes:

- There is no `@scope` shorthand for reusable workflows — the
  `.github/workflows/<file>.yaml` path and trailing `@<ref>` are required.
- This repo is **private**, so consuming repos need access:
  Settings → Actions → General → Access → "Accessible from repositories
  owned by the user".
- `@main` tracks the tip. Pin to a tag or SHA (`@v1`) for stability.
- AWS-touching workflows assume the caller grants `id-token: write` and
  use OIDC via the `aws_role` input — no long-lived keys.

## Workflows

| Workflow                  | Purpose                                                                         |
| ------------------------- | ------------------------------------------------------------------------------- |
| `nodejs-cdk-ci.yaml`      | Install → build → test → `cdk synth` for a Node CDK package.                    |
| `nodejs-ci.yaml`          | Typecheck + lint for a plain Node package (no CDK).                             |
| `nodejs-cdk-diff.yaml`    | `cdk diff` one stack, rendered into the job summary. Ungated.                   |
| `nodejs-cdk-deploy.yaml`  | `cdk deploy` one stack, gated by a GitHub `environment`.                        |
| `nodejs-cdk-destroy.yaml` | `cdk destroy` one stack, then assert it's gone.                                 |
| `eas-build.yaml`          | Submit an Expo EAS build (`--no-wait`); collect the APK from the EAS dashboard. |

### `nodejs-cdk-ci.yaml`

| Input               | Required | Default        | Notes                                     |
| ------------------- | -------- | -------------- | ----------------------------------------- |
| `working-directory` | no       | `.`            | Dir holding `package.json` + `cdk.json`.  |
| `node-version`      | no       | `24.x`         |                                           |
| `aws-region`        | no       | `eu-central-1` | Region for `cdk synth` (no creds needed). |

Runs `npm ci` → `npm run build` → `npm test` → `cdk synth`.

### `nodejs-ci.yaml`

| Input               | Required | Default | Notes                |
| ------------------- | -------- | ------- | -------------------- |
| `working-directory` | no       | `.`     |                      |
| `node-version`      | no       | `24.x`  |                      |
| `typecheck`         | no       | `true`  | Runs `tsc --noEmit`. |
| `lint`              | no       | `true`  | Runs `npm run lint`. |

### `nodejs-cdk-diff.yaml`

| Input               | Required | Default        | Notes                                                                                      |
| ------------------- | -------- | -------------- | ------------------------------------------------------------------------------------------ |
| `aws_role`          | **yes**  | —              | OIDC role ARN to assume. CDK chain-assumes the target account's bootstrap roles from here. |
| `stack_name`        | **yes**  | —              | Exact CDK stack name.                                                                      |
| `aws_region`        | no       | `eu-central-1` |                                                                                            |
| `stack_prefix`      | no       | `""`           | Value for `CDK_STACK_PREFIX` (e.g. `pr42-`).                                               |
| `working-directory` | no       | `.`            |                                                                                            |
| `node-version`      | no       | `24.x`         |                                                                                            |

No `environment:` — runs ungated so a reviewer reads the diff before
approving a gated deploy.

### `nodejs-cdk-deploy.yaml`

| Input               | Required | Default        | Notes                                              |
| ------------------- | -------- | -------------- | -------------------------------------------------- |
| `aws_role`          | **yes**  | —              | OIDC role ARN to assume.                           |
| `stack_name`        | **yes**  | —              | Exact CDK stack name.                              |
| `environment`       | **yes**  | —              | GitHub environment = the approval gate.            |
| `aws_region`        | no       | `eu-central-1` |                                                    |
| `stack_prefix`      | no       | `""`           | Value for `CDK_STACK_PREFIX`.                      |
| `outputs-artifact`  | no       | `""`           | If set, upload `cdk-outputs.json` under this name. |
| `working-directory` | no       | `.`            |                                                    |
| `node-version`      | no       | `24.x`         |                                                    |

### `nodejs-cdk-destroy.yaml`

| Input               | Required | Default        | Notes                                   |
| ------------------- | -------- | -------------- | --------------------------------------- |
| `aws_role`          | **yes**  | —              | OIDC role ARN to assume.                |
| `stack_name`        | **yes**  | —              | Exact CDK stack name.                   |
| `environment`       | **yes**  | —              | GitHub environment for the destroy job. |
| `aws_region`        | no       | `eu-central-1` |                                         |
| `stack_prefix`      | no       | `""`           | Match the target's `CDK_STACK_PREFIX`.  |
| `working-directory` | no       | `.`            |                                         |
| `node-version`      | no       | `24.x`         |                                         |

### `eas-build.yaml`

| Input               | Required | Default      | Notes                                                                      |
| ------------------- | -------- | ------------ | -------------------------------------------------------------------------- |
| `working-directory` | no       | `mobile`     | Dir holding the Expo app.                                                  |
| `platform`          | no       | `android`    | `android` \| `ios`.                                                        |
| `profile`           | no       | `production` | EAS build profile (from eas.json); its `env` block carries per-env values. |
| `config-artifact`   | no       | `""`         | Optional artifact extracted into `<working-directory>/config/` first.      |
| `node-version`      | no       | `24.x`       |                                                                            |

| Secret       | Required | Notes                                         |
| ------------ | -------- | --------------------------------------------- |
| `EXPO_TOKEN` | **yes**  | Expo access token (expo.dev → access tokens). |
