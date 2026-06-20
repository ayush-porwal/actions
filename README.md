# actions

Reusable **GitHub Actions workflows**, called at the _job_ level from a consuming repo. Every workflow takes **caller-provided paths and names** (`install-directory`, `working-directory`, `lockfile-path`, `npm-workspace`, etc.) so the same actions work for standalone packages and npm-workspace monorepos — nothing assumes a fixed project layout.

## Usage

Reference a workflow by its full path plus a git ref:

```yaml
jobs:
  build:
    uses: ayush-porwal/actions/.github/workflows/nodejs-cdk-ci.yaml@main
    with:
      install-directory: .
      working-directory: infra
      lockfile-path: package-lock.json
      npm-workspace: my-infra-package
      node-version: "24.x"
```

### Standalone package (legacy layout)

Each package has its own `package-lock.json`:

```yaml
with:
  working-directory: infra
  # install-directory defaults to working-directory
  # lockfile-path defaults to infra/package-lock.json
```

### npm-workspace monorepo

Single root lockfile; CDK lives in a subdirectory:

```yaml
with:
  install-directory: .
  working-directory: infra
  lockfile-path: package-lock.json
  npm-workspace: my-infra-package
```

Notes:

- There is no `@scope` shorthand for reusable workflows — the
  `.github/workflows/<file>.yaml` path and trailing `@<ref>` are required.
- This repo must be accessible from consuming repos (matching visibility).
- `@main` tracks the tip. Pin to a tag or SHA (`@v1`) for stability.
- AWS-touching workflows assume the caller grants `id-token: write` and
  use OIDC via the `aws_role` input — no long-lived keys.

## Workflows

| Workflow                     | Purpose                                                              |
| ---------------------------- | -------------------------------------------------------------------- |
| `nodejs-cdk-ci.yaml`         | Install → lint (opt) → build → test → `cdk synth`.                 |
| `nodejs-ci.yaml`             | Typecheck + lint for a plain Node package.                           |
| `nodejs-cdk-diff.yaml`       | `cdk diff` one stack, rendered into the job summary. Ungated.          |
| `nodejs-cdk-diff-deploy.yaml`| `cdk diff` → summary → `cdk deploy` in one job (ungated envs).       |
| `nodejs-cdk-deploy.yaml`     | `cdk deploy` one stack, gated by a GitHub `environment`.             |
| `nodejs-cdk-destroy.yaml`    | `cdk destroy` one stack, then assert it's gone.                      |
| `eas-build.yaml`             | Submit an Expo EAS build (`--no-wait`).                              |

### Common install inputs

These appear on most Node workflows:

| Input               | Required | Default | Notes                                                                 |
| ------------------- | -------- | ------- | --------------------------------------------------------------------- |
| `install-directory` | no       | `""`    | Where `npm ci` runs. Empty → same as `working-directory`.             |
| `working-directory` | no       | `.`     | Where CDK / EAS commands run.                                         |
| `lockfile-path`     | no       | `""`    | npm cache key. Empty → `<install-directory>/package-lock.json`.       |
| `npm-workspace`     | no       | `""`    | When set, scripts run as `npm run <script> --workspace=<name>`.      |
| `node-version`      | no       | `24.x`  |                                                                       |

### `nodejs-cdk-ci.yaml`

| Input           | Required | Default | Notes                          |
| --------------- | -------- | ------- | ------------------------------ |
| `lint`          | no       | `false` | Run `npm run lint`.            |
| `format-check`  | no       | `false` | Run `npm run format:check`.    |
| `aws-region`    | no       | `eu-central-1` | Region for dummy-cred synth. |

Plus the common install inputs above.

### `nodejs-ci.yaml`

| Input          | Required | Default | Notes                    |
| -------------- | -------- | ------- | ------------------------ |
| `typecheck`    | no       | `true`  | Run typecheck.           |
| `lint`         | no       | `true`  | Run lint.                |
| `format-check` | no       | `false` | Run `format:check`.      |

Plus the common install inputs above.

### `nodejs-cdk-diff.yaml` / `nodejs-cdk-deploy.yaml` / `nodejs-cdk-destroy.yaml`

| Input          | Required | Default        | Notes                                              |
| -------------- | -------- | -------------- | -------------------------------------------------- |
| `aws_role`     | **yes**  | —              | OIDC role ARN.                                     |
| `stack_name`   | **yes**  | —              | Exact CDK stack name.                              |
| `environment`  | deploy/destroy only | — | GitHub environment (approval gate).     |
| `stack_prefix` | no       | `""`           | Value for `CDK_STACK_PREFIX`.                      |
| `outputs-artifact` | deploy only | `""`    | Upload `cdk-outputs.json` under this name.         |
| `aws_region`   | no       | `eu-central-1` |                                                    |

Plus the common install inputs above.

`nodejs-cdk-diff.yaml` has no `environment:` — runs ungated so a reviewer reads the diff before approving a gated deploy.

### `nodejs-cdk-diff-deploy.yaml`

Same inputs as diff + deploy combined. Runs diff → job summary → deploy in **one job** (one install). Use for ungated environments (e.g. per-PR sandboxes). Keep separate diff/deploy for staging/prod where reviewers must read the plan before approving.

### `eas-build.yaml`

| Input               | Required | Default    | Notes                                                       |
| ------------------- | -------- | ---------- | ----------------------------------------------------------- |
| `eas-project-root`  | no       | `""`       | EAS upload root. Empty → `<workspace>/<working-directory>`. Set to monorepo root when needed. |
| `config-directory`  | no       | `""`       | Where `config-artifact` is extracted. Empty → `<working-directory>/config`. |
| `platform`          | no       | `android`  | `android` \| `ios`.                                         |
| `profile`           | no       | `production` | EAS build profile.                                        |
| `config-artifact`   | no       | `""`       | Optional artifact extracted before building.                |

Plus the common install inputs above (except `npm-workspace`).

| Secret       | Required | Notes                                         |
| ------------ | -------- | --------------------------------------------- |
| `EXPO_TOKEN` | **yes**  | Expo access token (expo.dev → access tokens). |
